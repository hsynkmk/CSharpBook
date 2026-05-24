# 14 — Runtime, CLR & JIT — Coding Questions

> Predict / explain. (Concepts: [14-Runtime-CLR-JIT.md](14-Runtime-CLR-JIT.md))

---

### Q1 — Why is the first call slower?
```csharp
var sw = Stopwatch.StartNew(); Work(); Console.WriteLine(sw.ElapsedMilliseconds);   // run A
sw.Restart();                  Work(); Console.WriteLine(sw.ElapsedMilliseconds);   // run B
```
<details><summary>Answer</summary>

Run A is **slower** — the method is **JIT-compiled** on first call (IL → native). Run B uses the already-compiled (tier-0, later tier-1) code. This "warmup" is why BenchmarkDotNet does warmup iterations and why naive timing misleads. ([21-Deployment-Perf-Tooling.md](21-Deployment-Perf-Tooling.md))
</details>

---

### Q2 — Is this interpreted?
```csharp
int Add(int a, int b) => a + b;
```
<details><summary>Answer</summary>

**No.** C# compiles to **IL** (in the DLL); at runtime the **JIT** compiles IL to **native machine code** before execution. It's compiled, not interpreted (unlike default CPython). (`dynamic`/expression interpretation are exceptions, not the norm.)
</details>

---

### Q3 — What breaks under Native AOT?
```csharp
var type = Type.GetType("MyApp.Plugins." + name);
var instance = Activator.CreateInstance(type!);
var json = JsonSerializer.Serialize(instance);   // reflection-based
```
<details><summary>Answer</summary>

All of it is **AOT-hostile**: runtime type lookup + `Activator.CreateInstance` + reflection-based serialization can be **trimmed away** or unsupported → runtime failures. **Fix:** source generators (System.Text.Json source-gen), explicit registration, `[DynamicDependency]` annotations. AOT needs statically-analyzable code.
</details>

---

### Q4 — Tiered compilation behavior
```csharp
// A hot loop called millions of times. What does the runtime do?
for (long i = 0; i < 1_000_000_000; i++) HotMethod();
```
<details><summary>Answer</summary>

`HotMethod` starts as **tier-0** (quick, less-optimized) for fast startup, then the runtime detects it's **hot** and recompiles it as **tier-1** (fully optimized) in the background. **OSR** can even upgrade a long-running loop mid-execution. **Dynamic PGO** uses tier-0 profiling to optimize tier-1.
</details>

---

### Q5 — Framework-dependent vs self-contained
```bash
dotnet publish -c Release                       # (a)
dotnet publish -c Release -r linux-x64 --self-contained   # (b)
```
<details><summary>Answer</summary>

(a) **framework-dependent** — small, requires the matching .NET runtime installed on the target. (b) **self-contained** — bundles the runtime (larger, needs a RID), runs on a machine without .NET. Choose by whether you control the runtime on the target.
</details>

---

### Q6 — Generic code generation (senior)
```csharp
var a = new List<int>();      // value type
var b = new List<string>();   // reference type
var c = new List<DateTime>(); // value type
```
<details><summary>Answer</summary>

The JIT generates **separate specialized native code** for `List<int>` and `List<DateTime>` (value types — different layouts, box-free), but **shares one** code body for all reference types (`List<string>`, `List<object>`, …) since references are uniform pointers. Explains why value-type generics avoid boxing but increase code size.
</details>

---

### Q7 — When does R2R help (senior)?
```csharp
// A serverless function with strict cold-start limits, heavy reflection (can't AOT).
```
<details><summary>Answer</summary>

Use **ReadyToRun** — it precompiles IL to native **alongside** the IL at publish, cutting JIT work at startup (faster cold start) **without** AOT's restrictions (reflection still works, JIT still re-optimizes hot paths). It's the no-compromise startup win when AOT isn't viable.
</details>
