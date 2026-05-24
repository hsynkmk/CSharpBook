# Chapter 13 — I/O & Serialization

> Files, streams, pipelines, JSON, XML, encodings, and globalization. Everything that crosses the boundary between your program and the outside world.

**Prerequisites**: Chapters 02 (OOP), 09 (Memory).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-FileSystem.md](01-FileSystem.md) | `File`, `Directory`, `Path`, `FileInfo`/`DirectoryInfo`, async I/O methods, atomic writes via temp+rename. |
| [02-Streams.md](02-Streams.md) | `Stream` base class, `FileStream`, `MemoryStream`, `BufferedStream`, async I/O, position/seek semantics, ownership. |
| [03-Pipelines.md](03-Pipelines.md) | `System.IO.Pipelines`, `PipeReader`/`PipeWriter`, when high-perf network code wants pipelines instead of streams. |
| [04-TextReadersWriters.md](04-TextReadersWriters.md) | `StreamReader`/`StreamWriter`, `StringReader`/`StringWriter`, encoding traps, `Regex` here too. |
| [05-SystemTextJson.md](05-SystemTextJson.md) | `JsonSerializer`, `JsonNode`/`JsonDocument`, options, naming policies, custom converters, source-generated serialization. |
| [06-XmlAndYaml.md](06-XmlAndYaml.md) | `XDocument` / LINQ to XML, `XmlSerializer`, `DataContractSerializer`, third-party YAML options. |
| [07-Encoding.md](07-Encoding.md) | `Encoding` class, UTF-8 vs UTF-16, BOM, decoding errors, choosing the right encoding. |
| [08-Globalization.md](08-Globalization.md) | `CultureInfo`, invariant culture, parsing dates and numbers across locales, normalization. |
| [Questions.md](Questions.md) | ~20 questions. |
| [Coding.md](Coding.md) | ~12 problems: streaming large JSON, atomic file write, custom JSON converter. |

→ Begin: [01-FileSystem.md](01-FileSystem.md)
