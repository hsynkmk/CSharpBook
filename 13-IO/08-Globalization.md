# Globalization

## What it is

Globalization is the handling of **culture-specific** formatting and parsing: dates, numbers, currencies, string comparison, casing, and sorting. The same data renders differently per locale.

```csharp
double price = 1234.56;
price.ToString("C", new CultureInfo("en-US"));   // $1,234.56
price.ToString("C", new CultureInfo("de-DE"));   // 1.234,56 €
price.ToString("C", new CultureInfo("fr-FR"));   // 1 234,56 €
```

Note: `en-US` uses `.` for decimals and `,` for thousands; `de-DE` reverses them. This is the source of countless parsing bugs.

---

## `CultureInfo`

Represents a culture (language + region): `"en-US"`, `"de-DE"`, `"ja-JP"`, etc.

```csharp
using System.Globalization;

var us = new CultureInfo("en-US");
var de = CultureInfo.GetCultureInfo("de-DE");   // cached instance
var current = CultureInfo.CurrentCulture;        // thread's culture (formatting)
var ui = CultureInfo.CurrentUICulture;           // thread's culture (resource lookup / translations)
```

Two ambient cultures per thread:
- **`CurrentCulture`** — affects formatting/parsing (numbers, dates).
- **`CurrentUICulture`** — affects which localized resource strings load.

```csharp
// Set for the whole app (modern .NET)
CultureInfo.DefaultThreadCurrentCulture = new CultureInfo("en-US");
CultureInfo.DefaultThreadCurrentUICulture = new CultureInfo("en-US");
```

---

## The golden rule: invariant for machine, current for humans

| Scenario | Culture |
|---|---|
| Saving to file / DB / JSON / wire | **`CultureInfo.InvariantCulture`** |
| Parsing config, IDs, machine data | **`CultureInfo.InvariantCulture`** |
| Displaying to a user | `CurrentCulture` (their locale) |
| Logs (mostly machine-read) | `InvariantCulture` |

```csharp
// ✓ — persist with invariant culture (stable, locale-independent)
string saved = value.ToString(CultureInfo.InvariantCulture);
double loaded = double.Parse(saved, CultureInfo.InvariantCulture);

// ✓ — display with the user's culture
string shown = value.ToString("N2", CultureInfo.CurrentCulture);
```

`InvariantCulture` is a culture-neutral baseline (English-like, `.` decimals). Using it for storage means data round-trips regardless of the machine's locale.

---

## The classic bug

```csharp
// On a German machine (CurrentCulture = de-DE)
double d = 1234.56;
string s = d.ToString();          // "1234,56"  ← comma decimal!
File.WriteAllText("data.txt", s);

// Later, on a US machine
double back = double.Parse(File.ReadAllText("data.txt"));   // 123456.0  ← WRONG (comma ignored)
```

The number was written with a German decimal comma, then parsed on a US machine that treats `,` as a thousands separator. **Data corruption.**

Fix: always use `InvariantCulture` for persistence.

```csharp
File.WriteAllText("data.txt", d.ToString(CultureInfo.InvariantCulture));   // "1234.56"
double back = double.Parse(text, CultureInfo.InvariantCulture);            // 1234.56 ✓
```

---

## Parsing numbers

```csharp
// Always specify culture and style for robust parsing
double d = double.Parse("1,234.56", NumberStyles.Number, CultureInfo.InvariantCulture);

// Safe parse
if (decimal.TryParse(input, NumberStyles.Currency, CultureInfo.CurrentCulture, out var amount)) { ... }

// Integer with thousands separators
int n = int.Parse("1,000", NumberStyles.AllowThousands, CultureInfo.InvariantCulture);
```

`NumberStyles` controls what's allowed (leading sign, thousands separators, currency symbol, etc.). Be explicit for untrusted input.

---

## Formatting numbers

```csharp
1234.5.ToString("N2", inv);    // 1,234.50  (number, 2 decimals)
1234.5.ToString("C", us);      // $1,234.50 (currency)
0.156.ToString("P1", inv);     // 15.6%     (percent)
255.ToString("X");             // FF        (hex)
1234.ToString("D6");           // 001234    (padded integer)
1234.5.ToString("F2", inv);    // 1234.50   (fixed, no thousands)
```

Standard format specifiers (`N`, `C`, `P`, `F`, `E`, `D`, `X`) + custom patterns (`#,##0.00`).

---

## Dates and times

```csharp
var dt = new DateTime(2026, 5, 22, 14, 30, 0, DateTimeKind.Utc);

dt.ToString("d", us);                  // 5/22/2026
dt.ToString("d", de);                  // 22.05.2026
dt.ToString("O");                      // 2026-05-22T14:30:00.0000000Z (round-trip ISO 8601)
dt.ToString("R");                      // RFC 1123: Fri, 22 May 2026 14:30:00 GMT

// Parsing — specify format and culture
DateTime parsed = DateTime.ParseExact("22.05.2026", "dd.MM.yyyy", de);
DateTime iso = DateTime.Parse("2026-05-22T14:30:00Z", CultureInfo.InvariantCulture,
    DateTimeStyles.RoundtripKind);
```

### Persisting dates

Use the **round-trip "O" format** (ISO 8601) with invariant culture for storage:

```csharp
string saved = dt.ToString("O", CultureInfo.InvariantCulture);   // unambiguous, sortable
DateTime back = DateTime.Parse(saved, CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);
```

Prefer `DateTimeOffset` over `DateTime` for storage — it carries the offset and avoids `DateTimeKind` ambiguity. See [DotNetBook / Chapter on date-time] for the full DateTime vs DateTimeOffset discussion.

---

## String comparison and culture

String comparison is culture-sensitive — a frequent subtle bug.

```csharp
// ⚠ — culture-sensitive comparison (varies by locale, e.g. Turkish 'i' problem)
"file".ToUpper();                              // uses CurrentCulture
str1.Equals(str2, StringComparison.CurrentCulture);

// ✓ — for identifiers, paths, protocol tokens: use Ordinal (byte-wise, fast, stable)
str1.Equals(str2, StringComparison.Ordinal);
str1.Equals(str2, StringComparison.OrdinalIgnoreCase);
"file".ToUpperInvariant();                     // stable casing
```

### The Turkish-I problem

In Turkish culture, `"i".ToUpper()` is `"İ"` (dotted capital I), not `"I"`. Code that uppercases identifiers culture-sensitively breaks on Turkish systems:

```csharp
// ⚠ — fails on tr-TR: "file".ToUpper() != "FILE"
if (ext.ToUpper() == "TXT") { ... }

// ✓ — ordinal/invariant for non-linguistic comparison
if (ext.Equals("txt", StringComparison.OrdinalIgnoreCase)) { ... }
```

**Rule**: use `Ordinal`/`OrdinalIgnoreCase` for identifiers, file extensions, keys, protocol strings. Use `CurrentCulture` only for user-facing sorting/display.

---

## Sorting

```csharp
var words = new[] { "apple", "Äpfel", "banana" };

// Culture-aware sort (linguistic — what users expect in their language)
Array.Sort(words, StringComparer.Create(CultureInfo.CurrentCulture, ignoreCase: false));

// Ordinal sort (byte-wise — fast, deterministic, but "Z" < "a")
Array.Sort(words, StringComparer.Ordinal);
```

User-visible lists → culture comparer. Internal/deterministic ordering → ordinal.

---

## Normalization

The same visible string can have different byte representations (e.g., `é` as one code point vs `e` + combining accent). Normalize before comparing:

```csharp
string a = "café";              // U+00E9 (precomposed)
string b = "café";        // e + combining acute

a == b;                                          // false! different code points
a.Normalize() == b.Normalize();                  // true (both → NFC)
```

`Normalize(NormalizationForm.FormC)` (default) composes; `FormD` decomposes. Normalize user input (especially for security comparisons, usernames) to avoid homograph issues.

---

## Globalization-invariant mode

For containers/microservices that don't need culture data, you can run in invariant mode to shrink the runtime and avoid ICU dependencies:

```xml
<PropertyGroup>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

In this mode, all cultures behave like `InvariantCulture`. Good for backend services with no localization needs; smaller image, faster startup, no ICU. **Don't** use it if you display localized content.

---

## Common bugs

### Persisting with current culture

The #1 globalization bug. Always `InvariantCulture` for storage/wire. (See classic bug above.)

### Culture-sensitive comparison for identifiers

Use `Ordinal`. The Turkish-I problem and performance both argue for it.

### `DateTime.Parse` without format/culture

```csharp
DateTime.Parse("01/02/2026");   // ⚠ — Jan 2 (US) or Feb 1 (EU)? Ambiguous!
```

Use `ParseExact` with an explicit format + culture, or ISO 8601.

### Forgetting Unicode normalization

User input from different keyboards/OSes may not be byte-equal even when visually identical. Normalize before security-sensitive comparisons.

---

## Performance notes

- `Ordinal` comparison is faster than culture-aware (no collation tables).
- `CultureInfo.GetCultureInfo(name)` returns cached instances; `new CultureInfo(name)` allocates.
- `InvariantGlobalization` mode reduces startup and memory for services.
- Caching formatted strings for repeated values helps in hot paths.

---

## When to use what

| Operation | Culture |
|---|---|
| Store/load number, date | `InvariantCulture` (+ ISO format for dates) |
| Display to user | `CurrentCulture` |
| Compare identifiers/keys/extensions | `StringComparison.Ordinal[IgnoreCase]` |
| Sort user-visible list | culture-aware `StringComparer` |
| Uppercase for normalization | `ToUpperInvariant` |
| Security comparison of input | `Normalize()` + `Ordinal` |

---

## Summary

- Globalization handles culture-specific formatting/parsing of numbers, dates, strings.
- **Invariant culture for machines** (storage, wire); **current culture for humans** (display).
- The classic bug: persisting with current culture corrupts data across locales — always use `InvariantCulture`.
- Use `Ordinal`/`OrdinalIgnoreCase` for identifiers (and to dodge the Turkish-I problem); culture-aware only for user-facing sort/display.
- Use ISO 8601 ("O") + `DateTimeOffset` for date storage; `ParseExact` for parsing.
- Normalize Unicode before sensitive comparisons.
- `InvariantGlobalization` mode for non-localized services.

→ Next: [Questions.md](Questions.md)
