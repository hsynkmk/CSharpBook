# String Interning

## What it is

The CLR maintains an **intern pool** — a process-wide table of unique string instances. When you write a string literal in source code, the compiler emits a reference to the interned copy. Equal literals share the same heap object.

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b));   // true — same interned string

string c = string.Intern("he" + "llo");
Console.WriteLine(ReferenceEquals(a, c));   // true — explicit intern produced the same
```

Runtime-built strings (from `ReadLine`, `Substring`, `new string`, file I/O) are **not** interned by default. They're regular heap strings.

Used to:
- **Save memory** for frequently-occurring strings (e.g., common keys).
- **Speed up comparisons** when you know strings are interned (`ReferenceEquals` is O(1)).

Often more curiosity than necessity. Modern code rarely uses Intern directly.

---

## Why it exists

Constants like `"yes"`, `"no"`, `"GET"`, `"application/json"` appear thousands of times in a program. Interning ensures only ONE heap string per distinct literal — significant memory savings for large codebases with many literal strings.

Reference comparison after interning is O(1) — just a pointer compare. Value comparison would be O(length).

For program startup data tables (config keys, enum-string maps), interning makes lookups faster too.

---

## Automatic interning of literals

The compiler emits all string literals into the assembly's **metadata table**. At runtime, when a literal is loaded, the runtime looks it up in the intern pool:

- If present, returns the existing reference.
- If absent, adds the new string and returns it.

So:
```csharp
string a = "hello";   // a refers to the intern pool's "hello"
string b = "hello";   // b refers to the SAME interned string
```

This happens for ALL string literals automatically. No opt-in needed.

Across all loaded assemblies in a process, "hello" literals are unified to one instance.

---

## `string.Intern` and `string.IsInterned`

Explicitly intern a runtime string:

```csharp
string s = Console.ReadLine() ?? "";   // not interned
string interned = string.Intern(s);     // returns the interned copy (existing or new)
```

If a string equal to `s` is already in the pool, returns that. Else adds `s` to the pool and returns it.

Check without adding:

```csharp
string? maybe = string.IsInterned(s);   // null if not interned; the interned instance otherwise
```

`IsInterned` doesn't add to the pool — useful for inspection.

---

## When interning helps

### Many copies of the same runtime string

```csharp
// 10K rows, each with a "Type" string repeated many times
foreach (var row in rows) {
    row.Type = string.Intern(row.Type);
}
```

If "Type" has only ~10 distinct values, after interning you have 10 strings instead of 10K. Memory saved.

### Hot key comparisons

```csharp
string key = string.Intern(receivedKey);
foreach (var item in list) {
    if (ReferenceEquals(item.Key, key)) { ... }   // O(1) instead of O(len)
}
```

If both `item.Key` and `key` are interned, reference equality works AND is faster than `string.Equals`.

But: most string equality in .NET is already fast (the JIT optimizes ordinal compares aggressively). The win is small in practice.

---

## When interning HURTS

### Interned strings live forever

The intern pool is a GC root. Strings in the pool are NEVER collected (until the process exits).

```csharp
foreach (var row in millionRows) {
    string.Intern(row.UniqueId);   // ⚠ — million unique IDs interned, never freed
}
```

Process memory grows continuously. **The intern pool can leak memory if you intern unbounded data.**

Only intern strings that are:
- Known to be a small finite set.
- Worth keeping alive for the entire process.

For unbounded data, **don't** intern.

### Per-intern cost

`string.Intern(s)` does a hash lookup + possible insertion. Not free — ~50-100 ns. For one-off strings, not worth it.

---

## Alternatives to manual interning

For modern code, prefer:

### `StringPool` (third-party, e.g., CommunityToolkit)

```csharp
// CommunityToolkit.HighPerformance
StringPool.Shared.GetOrAdd(span);
```

Like Intern but **bounded** — won't grow without limit. Use this for "intern-like" pooling of dynamic strings.

### `FrozenDictionary` / `FrozenSet` (.NET 8+)

If you need to look up "is this string in my known set?", use a Frozen collection. No GC root, no unbounded growth, fast lookups:

```csharp
private static readonly FrozenSet<string> KnownTypes = new[] {
    "A", "B", "C"
}.ToFrozenSet();

bool known = KnownTypes.Contains(received);
```

### `Encoding.UTF8.GetString(...)` with `string.Create`

For building short strings from spans efficiently. Doesn't intern but doesn't have the pool's pitfalls.

---

## Internals — how the intern pool works

The CLR maintains a per-AppDomain hash table:
- Key: string contents (hashed).
- Value: the interned string instance.

When a literal is loaded:
1. Compute the hash.
2. Look up.
3. If found, return existing.
4. Else add the literal's instance (or a new one) and return it.

This is done lazily — only when the literal is referenced at runtime.

For `string.Intern(s)`:
1. Compute hash of s.
2. Look up.
3. If found, return existing.
4. Else, copy s (or use s itself if it's a fresh allocation) into the pool, return it.

The pool persists for the AppDomain lifetime (usually = process lifetime).

### Why it doesn't help for short-lived strings

If you intern a million unique strings, the pool grows to a million entries. They stay forever. Memory leak.

If you intern a hundred recurring strings (e.g., enum names), the pool stays small. Memory saved.

---

## Comparison: == vs ReferenceEquals vs Equals

For ANY two strings (interned or not):

```csharp
string a = "hello";
string b = "hello";
string c = new string("hello".ToCharArray());

a == b;                    // true (string overrides ==)
a == c;                    // true
ReferenceEquals(a, b);     // true (interned literals)
ReferenceEquals(a, c);     // false (c is a fresh allocation)
a.Equals(b);               // true
a.Equals(c);               // true
```

For interned strings, `ReferenceEquals` and `==` give the same answer. For non-interned, `==` does a value compare (slower but correct); `ReferenceEquals` may return false even for equal contents.

**Don't write `ReferenceEquals(a, b)` for string equality** unless you've proven both are interned. `==` is the safe choice.

---

## Surprising behaviors

### Compiler-folded literals

```csharp
string a = "hello";
string b = "hel" + "lo";
ReferenceEquals(a, b);   // true — compiler folds "hel" + "lo" at compile time
```

The compiler concatenates literal strings during compilation. The result is itself a literal in the metadata.

For runtime concatenation:
```csharp
string c = "hel";
string d = c + "lo";
ReferenceEquals(a, d);   // false — d is computed at runtime, not interned
```

### `string.Empty`

```csharp
ReferenceEquals(string.Empty, "");   // true
```

The empty string is interned as a singleton.

### Across assemblies

If two assemblies have the same string literal:
```csharp
// assembly1.dll
string x = "hello";
// assembly2.dll
string y = "hello";
ReferenceEquals(x, y);   // true — same interned string across the process
```

Process-wide pool.

### Constant fields

```csharp
public const string Greeting = "hello";
```

References to this constant become references to the interned string at compile time.

---

## CompilerGenerated string interning

The C# compiler interns:
- All string literals.
- All `const string` field references.
- Some expressions that the compiler can fold at compile time.

It does NOT intern:
- `new string(...)`.
- `string.Format(...)` results.
- Substring results.
- `Encoding.UTF8.GetString(...)` results.
- Concatenations involving variables.

Only literal-folded strings are auto-interned.

---

## Common bugs

### Using ReferenceEquals for string comparison

```csharp
if (ReferenceEquals(s, "hello")) { ... }   // ⚠ — works for literals, breaks for runtime strings
```

Use `==`:
```csharp
if (s == "hello") { ... }   // correct in all cases
```

The `s == "hello"` form does:
- Reference equality check (fast).
- Value equality fallback if they differ.

For interned literals on both sides, it's essentially as fast as ReferenceEquals.

### Interning unbounded data

```csharp
foreach (var row in dataFromUser) {
    string.Intern(row.Comment);   // ⚠ — pool grows unboundedly
}
```

User-supplied data has unknown cardinality. Interning makes pool grow forever. Memory leak.

Use `StringPool.Shared.GetOrAdd` (bounded LRU-like behavior) or just don't intern.

### Expecting intern to deduplicate at runtime automatically

```csharp
string a = ReadFromFile();
string b = ReadFromFile();   // same content as a
// a and b are DIFFERENT instances even with same content
```

Files, network, ToString results — none are auto-interned. If you want deduplication, you must explicitly call `string.Intern` or use a custom pool.

---

## `IsInterned` — the gotcha

```csharp
string s = Console.ReadLine() ?? "";
if (string.IsInterned(s) is not null) { /* it's interned */ }
```

`IsInterned` returns the interned instance if present, null otherwise. Doesn't add to the pool. Useful for inspection.

Most code doesn't use this. The intern pool is mostly transparent.

---

## Modern best practices

For 2026 C#:
- **Don't manually intern runtime strings** — usually a leak.
- **Use FrozenSet / FrozenDictionary** for known-string lookups.
- **For string deduplication in dynamic data**, use a third-party `StringPool` with bounded growth.
- **Trust the compiler** — literal interning is automatic and free.

The intern pool was more interesting in .NET Framework days when memory pressure was high. Modern .NET has better tools.

---

## When to actually intern

- A very small set of strings (< 100) used millions of times.
- Long-lived process where the pool's no-collection is acceptable.
- You've measured a real memory or perf win.

In production code, I've rarely seen a case where `string.Intern` was the right answer over a `Dictionary<string, T>` lookup.

---

## Performance

- Literal access: free (resolved at JIT time to direct reference).
- `string.Intern(s)`: ~50-100 ns (hash + lookup).
- `ReferenceEquals` on interned: ~1 ns (pointer compare).
- `==` on equal-length strings: usually 1-10 ns (SIMD ordinal compare).

For most code, `==` is fast enough that ReferenceEquals optimization isn't worth the discipline of interning.

---

## Summary

- The CLR has a process-wide intern pool for string deduplication.
- All string literals are automatically interned by the compiler.
- `string.Intern(s)` adds a runtime string to the pool.
- Interning saves memory ONLY when strings would be duplicated; otherwise it's a leak (interned strings never freed).
- For dynamic data, prefer `FrozenSet`, `StringPool`, or explicit dictionaries.
- `ReferenceEquals` on interned strings is fast but only correct if both sides are interned. Use `==` for safety.
- Largely a curiosity in modern .NET; rarely the right tool.

→ Next: [13-MemoryLeaks.md](13-MemoryLeaks.md)
