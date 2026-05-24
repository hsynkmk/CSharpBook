# Encoding

## What it is

An **encoding** is a mapping between abstract characters and the bytes that represent them on disk or wire. `System.Text.Encoding` is the class that performs this conversion.

```csharp
byte[] bytes = Encoding.UTF8.GetBytes("héllo");   // chars → bytes
string text = Encoding.UTF8.GetString(bytes);      // bytes → chars
```

Crucial mental model: **a `string` in .NET is UTF-16 in memory.** Encoding only matters when crossing the boundary to bytes (files, network, external systems).

```
"héllo" (string, UTF-16 internally)
   │  GetBytes(UTF-8)        │  GetBytes(UTF-16)         │  GetBytes(ASCII)
   ▼                          ▼                            ▼
68 C3 A9 6C 6C 6F (6 bytes)  68 00 E9 00 ... (10 bytes)  68 3F 6C 6C 6F ('?' for é)
```

---

## Why encoding matters

Get the encoding wrong and you get:
- **Mojibake** — `é` becomes `Ã©` (UTF-8 bytes read as Latin-1).
- **Data loss** — non-ASCII chars become `?` (encoding to ASCII).
- **Exceptions** — invalid byte sequences when strict.

Every byte↔text conversion has an implicit or explicit encoding. Mismatches between writer and reader are a classic bug source.

---

## The common encodings

| Encoding | Bytes/char | Notes |
|---|---|---|
| `Encoding.UTF8` | 1-4 (variable) | The universal default. ASCII-compatible. |
| `Encoding.Unicode` | 2 or 4 | UTF-16 LE. .NET's in-memory format. |
| `Encoding.BigEndianUnicode` | 2 or 4 | UTF-16 BE. |
| `Encoding.UTF32` | 4 (fixed) | Rare. |
| `Encoding.ASCII` | 1 | 0-127 only; loses non-ASCII. |
| `Encoding.Latin1` | 1 | ISO-8859-1, 0-255. (.NET 5+) |

**Default: always UTF-8** unless a system mandates otherwise. It's ASCII-compatible, compact for Western text, and the web standard.

```csharp
Encoding.UTF8           // recommended
Encoding.Unicode        // = UTF-16 LE
Encoding.Default        // ⚠ — historically the OS code page; now UTF-8 on .NET Core. Avoid relying on it.
```

---

## UTF-8 vs UTF-16

- **UTF-8** — variable width (1-4 bytes). ASCII = 1 byte. Compact for English/code. The file/wire standard.
- **UTF-16** — what .NET strings use in memory. 2 bytes for the BMP, 4 (surrogate pair) for the rest.

When you `File.WriteAllText`, the UTF-16 string is encoded to UTF-8 bytes (by default). Reading reverses it. The conversion is transparent but real work.

For interop, files, JSON, HTTP — use UTF-8. UTF-16 only matters internally or for specific legacy formats (some Windows APIs, .NET resource files).

---

## BOM (Byte Order Mark)

A BOM is a magic byte prefix declaring the encoding:

| Encoding | BOM bytes |
|---|---|
| UTF-8 | `EF BB BF` |
| UTF-16 LE | `FF FE` |
| UTF-16 BE | `FE FF` |

```csharp
// UTF-8 WITH BOM (default Encoding.UTF8 emits one when used by StreamWriter)
new UTF8Encoding(encoderShouldEmitUTF8Identifier: true);

// UTF-8 WITHOUT BOM (recommended for interop)
new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
```

### The BOM trap

A UTF-8 BOM is **optional and often harmful**:
- Shell scripts break (`#!/bin/sh` with a BOM prefix won't execute).
- Some JSON parsers reject a leading BOM.
- Concatenating BOM-prefixed files produces stray BOMs mid-stream.

**Recommendation**: write UTF-8 **without** BOM for interop. Read with BOM detection on (handles either).

```csharp
// Write without BOM
var noBom = new UTF8Encoding(false);
File.WriteAllText(path, content, noBom);
await File.WriteAllTextAsync(path, content, noBom);

// Read — StreamReader detects + strips BOM by default
using var reader = new StreamReader(path);   // detectEncodingFromByteOrderMarks: true by default
```

Note: `File.WriteAllText(path, content)` (no encoding arg) uses UTF-8 **without** BOM on modern .NET. Specifying `Encoding.UTF8` explicitly may add a BOM via some writer paths — be deliberate.

---

## Encoding/decoding errors

By default, invalid sequences are replaced with `�` (U+FFFD) or `?`. For strict validation:

```csharp
// Throw on invalid bytes/chars instead of silently replacing
var strict = new UTF8Encoding(
    encoderShouldEmitUTF8Identifier: false,
    throwOnInvalidBytes: true);

try {
    string s = strict.GetString(maybeCorruptBytes);
} catch (DecoderFallbackException) {
    // handle corrupt input
}
```

Use strict mode when you must detect corruption (data integrity); use replacement mode for best-effort display.

---

## Span-based and stream encoding

```csharp
// Span-based — allocation-friendly (.NET Core 2.1+)
ReadOnlySpan<char> chars = "hello";
Span<byte> bytes = stackalloc byte[Encoding.UTF8.GetByteCount(chars)];
int written = Encoding.UTF8.GetBytes(chars, bytes);

// Decode bytes → chars
Span<char> dest = stackalloc char[Encoding.UTF8.GetCharCount(bytes)];
int decoded = Encoding.UTF8.GetChars(bytes, dest);
```

For streaming (data arriving in chunks where a multi-byte char may split across reads), use `Encoder`/`Decoder` (stateful):

```csharp
Decoder decoder = Encoding.UTF8.GetDecoder();   // remembers partial sequences between calls
```

`StreamReader` uses a `Decoder` internally, which is why it correctly handles multi-byte chars spanning buffer boundaries.

---

## Legacy code pages

Code pages like Windows-1252 (`Latin1`) require registering a provider in .NET Core:

```csharp
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);   // from System.Text.Encoding.CodePages
var win1252 = Encoding.GetEncoding(1252);
```

.NET Core dropped most legacy code pages from the default set to reduce footprint. Register the provider only if you must read legacy-encoded data.

---

## UTF-8 string literals (`u8`)

C# 11+ lets you create UTF-8 bytes at compile time, skipping runtime encoding:

```csharp
ReadOnlySpan<byte> utf8 = "Content-Type: application/json"u8;
```

The bytes are baked into the assembly — no `GetBytes` call. Ideal for constant byte sequences (HTTP headers, protocol tokens). See [Chapter 10 §10](../10-AdvancedLanguage/10-UTF8StringLiterals.md).

---

## Common bugs

### Reading a file with the wrong encoding

```csharp
// File is Windows-1252; reading as UTF-8 corrupts non-ASCII
File.ReadAllText(path);   // ⚠ — assumes UTF-8

// Specify the actual encoding
File.ReadAllText(path, Encoding.GetEncoding(1252));
```

### Writing a BOM that breaks consumers

Covered above — use `new UTF8Encoding(false)`.

### Confusing byte length with char length

```csharp
string s = "héllo";
s.Length;                        // 5 (chars)
Encoding.UTF8.GetByteCount(s);   // 6 (bytes — é is 2 bytes in UTF-8)
```

Truncating a UTF-8 byte array at an arbitrary offset can split a multi-byte char. Truncate at char boundaries or decode first.

### Surrogate pairs and `string.Length`

```csharp
string emoji = "😀";
emoji.Length;   // 2 — it's a surrogate pair (2 UTF-16 code units), not 1 "character"
```

For user-perceived characters (grapheme clusters), use `System.Globalization.StringInfo` or `Rune` (`System.Text.Rune`) for Unicode scalar values.

```csharp
foreach (Rune rune in "héllo😀".EnumerateRunes()) { ... }   // iterate code points correctly
```

---

## Performance notes

- UTF-8 is compact and fast; it's the default for good reason.
- Use `GetByteCount`/`GetCharCount` to size buffers, then Span-based `GetBytes`/`GetChars` to avoid allocations.
- Reuse `Encoder`/`Decoder` for streaming.
- `u8` literals for constant byte sequences (zero runtime cost).
- Avoid `Encoding.Default` — its meaning shifted across .NET versions.

---

## When to use what

| Need | Use |
|---|---|
| Files, JSON, HTTP, most interop | UTF-8 (no BOM) |
| In-memory string (automatic) | UTF-16 (no choice) |
| ASCII-only protocol | `Encoding.ASCII` (validate!) |
| Legacy code page data | Register `CodePagesEncodingProvider` |
| Constant byte sequences | `"..."u8` literals |
| Iterate true code points | `Rune` / `EnumerateRunes` |

---

## Summary

- A `string` is UTF-16 in memory; encoding matters only at the byte boundary.
- Default to **UTF-8 without BOM** for files and interop.
- BOMs are often harmful — write without, read with detection.
- Wrong encoding → mojibake, data loss, or exceptions; always know your source encoding.
- `string.Length` counts UTF-16 code units, not characters — use `Rune` for code points.
- Use Span-based APIs and `u8` literals for performance.

→ Next: [08-Globalization.md](08-Globalization.md)
