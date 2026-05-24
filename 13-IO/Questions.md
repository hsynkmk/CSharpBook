# Chapter 13 — I/O & Serialization — Q & A

---

### Q1. `File.ReadAllLines` vs `File.ReadLines`?

`ReadAllLines` reads the entire file into a `string[]` (eager, all in memory). `ReadLines` returns `IEnumerable<string>` and reads lazily, one line at a time. Use `ReadLines` for large files to avoid loading everything.

---

### Q2. What does `Path.Combine` do that string concatenation doesn't?

It inserts the correct platform directory separator (`\` on Windows, `/` on Unix) and avoids double separators. `dir + "/" + file` breaks cross-platform and on edge cases. Always use `Path.Combine`.

---

### Q3. How do you write a file atomically?

Write to a temp file, then `File.Move(tmp, path, overwrite: true)`. Rename within a volume is atomic — readers see either the old or the complete new file, never a partial one. For durability, `Flush(flushToDisk: true)` before the rename.

---

### Q4. When use `FileInfo` over static `File` methods?

When querying the **same path repeatedly** — `FileInfo` caches metadata after first access, fewer syscalls. For one-off operations, static `File` methods are simpler.

---

### Q5. What's the contract of `Stream.Read`?

`Read(buffer, offset, count)` returns the number of bytes **actually read**, which may be **less than requested**; 0 means end of stream. You must loop until you've read enough — or use `ReadExactly` / `ReadAtLeast` / `CopyTo`. Assuming one `Read` returns everything is the #1 stream bug.

---

### Q6. `ToArray()` vs `GetBuffer()` on MemoryStream?

`ToArray()` returns a right-sized copy of the data. `GetBuffer()` returns the internal backing array (no copy) which may be larger than the data — you must use `Length` to know how much is valid.

---

### Q7. Why might a StreamReader close the caller's stream, and how to prevent it?

`StreamReader`/`StreamWriter` dispose their underlying stream on dispose by default. If the caller owns the stream, pass `leaveOpen: true` in the constructor so disposing the reader doesn't close the stream.

---

### Q8. What problem does System.IO.Pipelines solve over Stream?

Buffer management, partial-message handling, buffer copying, and back-pressure — all manual with `Stream`. Pipelines pools buffers, exposes `ReadOnlySequence<byte>` (multi-segment, no copying of leftovers), tracks consumed-vs-examined via `AdvanceTo`, and provides built-in back-pressure. Used for high-scale protocol parsing (Kestrel).

---

### Q9. What does `AdvanceTo(consumed, examined)` mean?

`consumed` = bytes fully processed (pipe can recycle). `examined` = bytes looked at (so the pipe waits for *new* data before returning the same buffer). If you examined everything but consumed only a complete-message prefix, the next `ReadAsync` waits for more data — no busy loop.

---

### Q10. Why is `ReadOnlySequence<byte>` multi-segment?

So the pipe never has to copy leftover bytes to the front of a buffer — it chains a new segment instead. Check `IsSingleSegment` for the fast path, or use `SequenceReader<byte>` to parse across segments.

---

### Q11. A `string` in .NET — what encoding is it in memory?

UTF-16. Encoding only matters at the byte boundary (files, network). `Encoding.UTF8.GetBytes` converts the in-memory UTF-16 string to UTF-8 bytes.

---

### Q12. UTF-8 with or without BOM for interop?

Without. A UTF-8 BOM (`EF BB BF`) is optional and breaks shell scripts, some JSON parsers, and file concatenation. Write with `new UTF8Encoding(false)`; read with BOM detection (StreamReader default) so either works.

---

### Q13. `string.Length` for "😀"?

2 — it's a surrogate pair (two UTF-16 code units), not one character. Use `Rune` / `EnumerateRunes` to iterate Unicode scalar values, or `StringInfo` for grapheme clusters.

---

### Q14. Why must you cache `JsonSerializerOptions`?

It caches type metadata internally. Creating a new instance per call rebuilds that cache every time — a major performance bug. Create one static instance and reuse it (it becomes immutable after first use).

---

### Q15. Is System.Text.Json case-sensitive by default?

Yes, on deserialize. JSON `{"name":...}` won't bind to a `Name` property unless `PropertyNameCaseInsensitive = true` or a camelCase policy is set. (ASP.NET's `Web` defaults are case-insensitive.)

---

### Q16. Does STJ serialize fields by default?

No — public **properties** only. Use `[JsonInclude]` on a field or `IncludeFields = true` in options to include fields.

---

### Q17. `JsonNode` vs `JsonDocument`?

`JsonNode`/`JsonObject`/`JsonArray` is a **mutable** DOM — read and modify. `JsonDocument`/`JsonElement` is **read-only**, uses pooled buffers (lower allocation, must be disposed), and is faster for inspection. Use `JsonDocument` when you only read; `JsonNode` when you mutate.

---

### Q18. Why use JSON source generation?

It emits serialization code at compile time — no reflection, AOT-safe, trimming-safe, faster startup, and often faster serialize. Required for NativeAOT. Declared via a `JsonSerializerContext` partial class with `[JsonSerializable]`.

---

### Q19. `XmlSerializer` caching gotcha?

`new XmlSerializer(type)` caches its generated assembly internally. But constructors taking extra args (like `XmlAttributeOverrides`) do **not** cache — repeated creation leaks dynamic assemblies. Always cache serializer instances.

---

### Q20. `XmlSerializer` (opt-out) vs `DataContractSerializer` (opt-in)?

`XmlSerializer` serializes all public read/write members unless `[XmlIgnore]` (opt-out). `DataContractSerializer` serializes only `[DataMember]`-marked members (opt-in). Use `XmlSerializer` for general XML, `DataContractSerializer` for WCF interop.

---

### Q21. Most common LINQ-to-XML bug?

Forgetting XML namespaces. `doc.Descendants("item")` returns nothing if the elements are in a namespace. Use `doc.Descendants(ns + "item")` with an `XNamespace`.

---

### Q22. Invariant vs current culture — when each?

**Invariant** for machine-readable data (storage, files, JSON, wire) so it round-trips regardless of locale. **Current** for displaying to users in their locale. Persisting with current culture is the classic globalization bug (e.g., German `1234,56` parsed wrongly on a US machine).

---

### Q23. What is the Turkish-I problem?

In Turkish culture, `"i".ToUpper()` yields `"İ"` (dotted), so culture-sensitive comparison of identifiers/extensions breaks. Use `StringComparison.Ordinal`/`OrdinalIgnoreCase` (or `ToUpperInvariant`) for non-linguistic comparisons.

---

### Q24. How should you persist a `DateTime`?

ISO 8601 round-trip format `"O"` with `InvariantCulture`, parsed back with `DateTimeStyles.RoundtripKind`. Prefer `DateTimeOffset` over `DateTime` for storage to carry the offset and avoid `DateTimeKind` ambiguity. Never persist with `DateTime.Parse` of a locale-formatted string.

---

### Q25. Why normalize Unicode before comparison?

The same visible string can have different code-point sequences (precomposed `é` vs `e` + combining accent) — `==` returns false despite looking identical. `Normalize()` (NFC) unifies them. Important for usernames and security-sensitive comparisons (homograph attacks).

---

### Q26. What is `InvariantGlobalization` mode?

A build setting (`<InvariantGlobalization>true</InvariantGlobalization>`) where all cultures behave like `InvariantCulture` — drops ICU dependency, smaller image, faster startup. Good for backend services with no localization; don't use it if you display localized content.

---

→ Next: [Coding.md](Coding.md)
