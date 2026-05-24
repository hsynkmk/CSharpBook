# Glossary

Every important term used in the book, defined precisely. Where a term has a dedicated chapter, the chapter is linked.

---

## A

**AOT (Ahead-of-Time compilation)** — Compiling C# directly to native machine code at publish time, eliminating the JIT. .NET 7+ supports **Native AOT**. Trade-off: faster startup, smaller deployment, but no runtime code generation (no reflection emit, no dynamic loading). See [Chapter 14 — Native AOT](14-InteropAOT/04-NativeAOT.md).

**Assembly** — The compiled output of a .NET project: a `.dll` or `.exe` containing IL code, metadata, and resources. The unit of versioning, deployment, and security.

**async / await** — C# keywords that let an asynchronous method pause without blocking a thread and resume when its awaited operation completes. The compiler rewrites `async` methods into a state machine. See [Chapter 8 — async/await](08-Concurrency/02-AsyncAwaitFundamentals.md).

**Attribute** — Metadata attached to a type, member, or assembly. Retrieved at runtime via reflection or at compile time by analyzers/source generators.

---

## B

**BCL (Base Class Library)** — The core library shipped with .NET: `System.*`, `System.Collections.*`, `System.IO.*`, etc. Available on every .NET runtime.

**Boxing** — Wrapping a value type instance in a heap-allocated object when assigned to an `object` or interface variable. Costs a heap allocation. See [Chapter 3 — Boxing](03-TypeSystem/07-BoxingUnboxing.md).

---

## C

**CLR (Common Language Runtime)** — The .NET virtual machine: JIT compiler, garbage collector, type system, security model. .NET 10's CLR is called CoreCLR.

**Closure** — A function combined with the variables it captures from its enclosing scope. In C#, closures are implemented as a compiler-generated class holding the captured variables. See [Chapter 5 — Closures](05-DelegatesEvents/04-Closures.md).

**Constraint (generic)** — A `where T : ...` clause limiting what types a generic parameter accepts. Constraints unlock operations on T (e.g., `where T : IComparable<T>` lets you call `CompareTo`).

---

## D

**DATAS (Dynamically Adapting To Application Sizes)** — .NET 8+ GC mode that auto-tunes heap thresholds per workload. **Default in .NET 10**. See [Chapter 9 — GC](09-MemoryPerformance/02-GarbageCollection.md).

**Delegate** — A type representing a method reference. Like a typed function pointer with multicast support. `Action`, `Func`, `Predicate` are predefined delegate types.

**Deferred execution** — A LINQ query doesn't run when defined; it runs when enumerated. `Select`/`Where` are deferred; `ToList`/`Count` materialize.

---

## E

**Escape analysis** — JIT optimization (improved in .NET 10) that determines whether an object's references escape the method. Non-escaping objects can be allocated on the stack instead of the heap.

**Expression tree** — A data-structure representation of code, used by IQueryable providers (EF Core) to translate C# lambdas into SQL.

**Extension method (or member, C# 14)** — A method (or property, operator, static — C# 14) defined outside a type but called as if it were a member. Implemented via a static class with `this` parameter, or via `extension` blocks (C# 14).

---

## F

**Finalizer** — A method (`~ClassName()`) run by the GC just before reclaiming an object. Used to release unmanaged resources when `Dispose` was not called. Adds GC overhead; prefer `SafeHandle`.

**field keyword (C# 14)** — Contextual keyword inside a property accessor that refers to the compiler-generated backing field. Lets you add logic without declaring a backing field manually.

**FrozenDictionary / FrozenSet (.NET 8)** — Read-only, immutable hash collections optimized for very fast lookups after construction. Build them once; never modify.

---

## G

**Garbage Collector (GC)** — The runtime component that automatically reclaims unreachable heap memory. .NET's GC is generational, tracing, mostly stop-the-world. See [Chapter 9 — GC](09-MemoryPerformance/02-GarbageCollection.md).

**Generation (GC)** — Heap partition by age: Gen0 (new), Gen1 (survived once), Gen2 (long-lived), LOH (≥85,000 bytes), POH (pinned).

**Generic** — A type or method parameterized over types: `List<T>`, `Dictionary<K,V>`, `static T Max<T>(...)`. Enables code reuse with type safety.

---

## H

**Heap** — Region of memory for reference-type instances and boxed values. Managed by the GC.

**Hoisting** — Compiler operation that promotes a local variable to a field of a closure class so it can be captured by lambdas.

---

## I

**IL (Intermediate Language)** — The platform-neutral bytecode produced by the C# compiler. Executed by the JIT (or compiled by AOT).

**IDisposable** — Interface with a single `Dispose()` method. The contract: callers invoke `Dispose()` (often via `using`) to release resources deterministically. See [Chapter 9 — IDisposable](09-MemoryPerformance/03-IDisposable.md).

**IAsyncDisposable** — Asynchronous variant: `ValueTask DisposeAsync()`. Use with `await using` for I/O-bound cleanup.

**ImmutableX** — Library collections (`System.Collections.Immutable`) that are persistent (operations return a new instance, sharing structure with the old).

**Interlocked** — Static class with atomic operations: `Increment`, `Decrement`, `CompareExchange`. Lock-free synchronization primitives.

---

## J

**JIT (Just-In-Time compiler)** — Compiles IL to machine code on first method invocation. .NET has tiered JIT: Tier 0 (fast compile, slow code) → Tier 1 (optimized).

---

## L

**LINQ (Language Integrated Query)** — Query syntax (`from x in xs where ... select ...`) and method syntax (`xs.Where(...).Select(...)`) for querying collections, databases, XML, etc.

**LOH (Large Object Heap)** — Heap partition for objects ≥85,000 bytes. Not compacted by default; expensive to allocate; collected only in Gen2.

---

## M

**Managed code** — Code executed under the CLR with GC, type safety, and exception handling.

**Memory&lt;T&gt;** — Heap-friendly counterpart to `Span<T>`. Can be stored in fields and crossed `await` boundaries. See [Chapter 9 — Memory](09-MemoryPerformance/06-Memory.md).

---

## N

**Native AOT** — See AOT.

**NRT (Nullable Reference Types)** — C# 8+ feature where reference types are non-null by default. `string` is non-null, `string?` is nullable. Compiler flow analysis warns of nullability bugs. See [Chapter 10 — NRT](10-AdvancedLanguage/01-NullableReferenceTypes.md).

---

## O

**Object initializer** — Syntax for setting properties on a freshly constructed object: `new Foo { X = 1, Y = 2 }`.

---

## P

**Pattern matching** — Construct (`is`, `switch` expression) for testing a value's shape — type, properties, list contents. Hugely expanded since C# 7. See [Chapter 3 — Pattern Matching](03-TypeSystem/09-PatternMatching.md).

**P/Invoke (Platform Invoke)** — Calling native (C/C++) functions from C# via `[DllImport]` or `[LibraryImport]`.

**POH (Pinned Object Heap)** — .NET 5+ heap partition for objects that are pinned (e.g., during P/Invoke). Avoids fragmenting Gen0/1/2.

**Primary constructor (C# 12)** — Class/struct syntax where constructor parameters live for the entire type body: `class Point(int x, int y) { ... }`.

---

## R

**Record (C# 9)** — Reference type with synthesized value equality, `ToString`, copy ctor, deconstruction. **Record struct** (C# 10) is the value-type variant.

**ref struct** — A struct that can only live on the stack. `Span<T>` is the canonical example. Cannot be a field of a class, cannot escape via async/iterators.

**Reflection** — Inspecting types, methods, fields, attributes at runtime. See [Chapter 12](12-Reflection/README.md).

---

## S

**SafeHandle** — Abstract class for wrapping native handles with built-in finalization and ref-counting. Preferred over manual `IntPtr` + finalizer.

**Sealed** — Modifier on a class (cannot be inherited) or method (cannot be overridden further). Helps JIT devirtualize calls.

**Span&lt;T&gt;** — Stack-only struct giving a typed view over contiguous memory (array, stackalloc, string slice). Allocation-free slicing.

**Source generator** — Compiler plugin that generates C# code at compile time. Use cases: JSON serialization, regex source-gen, dependency injection wiring.

**SynchronizationContext** — Abstraction for "where should this continuation run?" UI frameworks have one (UI thread); ASP.NET Core does NOT.

---

## T

**Task** — Type representing an asynchronous operation. `Task<T>` returns a value; `Task` returns nothing.

**Tiered compilation** — .NET 3.0+ JIT optimization strategy: code starts at Tier 0 (fast compile), promotes to Tier 1 (fully optimized) once it's hot.

**Trimming** — Build-time process that removes unreachable code from a published .NET app. Required for Native AOT; reduces deployment size.

**Tuple (ValueTuple)** — Lightweight value type carrying multiple values: `(int x, string y)`. Deconstructable: `var (a, b) = tuple;`.

---

## U

**Unmanaged code** — Native code (C/C++/assembly) not running under the CLR. Reached via P/Invoke.

**Unmanaged resource** — Memory, file handles, sockets, etc., allocated outside the GC's view. Released via `IDisposable` or finalizer.

**Unboxing** — Extracting the value type from a boxed object. Requires the runtime type to match exactly.

**Using statement / declaration** — `using (var x = ...) { ... }` or `using var x = ...;` — calls `Dispose` at end of scope.

---

## V

**ValueTask** — Lightweight `Task` alternative for hot paths where most calls complete synchronously (e.g., cache hits). Don't await twice.

**Variance** — How generic type parameters relate when types are substituted. `IEnumerable<out T>` is covariant; `Action<in T>` is contravariant.

**vtable (virtual method table)** — Runtime data structure mapping virtual method slots to actual implementations for a type. Drives runtime dispatch of `virtual`/`override` methods.

**volatile** — Field modifier preventing certain JIT/CPU reorderings of reads/writes. Use sparingly; `Interlocked` and `lock` are usually clearer.

---

## W

**`with` expression (C# 9)** — Non-destructive mutation on a record: `var p2 = p with { X = 5 };` — returns a copy with X changed.

**Worker thread** — Thread-pool thread used to run scheduled work, including async continuations.
