# Strings and Chars

## What it is

A `string` in C# is an **immutable sequence of UTF-16 code units**. It's a reference type — variables hold a reference to the heap-allocated `System.String` object. Strings are everywhere; getting them right is critical.

```csharp
string greeting = "Hello, World!";
char firstLetter = greeting[0];    // 'H'
int length = greeting.Length;       // 13
```

---

## Immutability

You **cannot change** a string after creating it. Every operation that "modifies" a string returns a new string:

```csharp
string s = "hello";
s.ToUpper();          // returns "HELLO" but doesn't change s
Console.WriteLine(s); // still "hello"

s = s.ToUpper();      // reassign s to the new string
Console.WriteLine(s); // "HELLO"
```

This is the **single most important fact** about C# strings. It's also why you should never build strings in a loop with `+`:

```csharp
// ❌ BAD — N allocations for N concatenations
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString();

// ✓ GOOD — one builder, one final allocation
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result = sb.ToString();
```

Why immutable? **Safety, threading, and string interning** all benefit. The downside is the building-up-strings pattern, solved by `StringBuilder`.

---

## String literals

### Regular strings

```csharp
string s = "hello";
string withQuote = "she said \"hi\"";     // escape with backslash
string withTab = "col1\tcol2\tcol3";
string newline = "first line\nsecond line";
string backslash = "C:\\Users\\me";        // each \\ is one backslash
```

### Verbatim strings — `@"..."`

Disable escape sequences. Great for file paths, regex, and multi-line text:

```csharp
string path = @"C:\Users\me\file.txt";    // no \\ doubling
string regex = @"\d{3}-\d{4}";            // \d not \\d
string multi = @"line one
line two
line three";                              // newlines included literally
```

To include a `"` in a verbatim string, double it: `@"she said ""hi"""`.

### Interpolated strings — `$"..."`

Embed expressions in `{ }`:

```csharp
string name = "Alice";
int age = 30;
string msg = $"{name} is {age} years old";   // "Alice is 30 years old"

// Format specifiers after a colon
double price = 19.99;
string s = $"Price: {price:F2}";              // "Price: 19.99"
string d = $"Date:  {DateTime.Now:yyyy-MM-dd}";

// Conditional inside
int n = 5;
string msg2 = $"{n} item{(n == 1 ? "" : "s")}";
```

Combine with `@` for both:

```csharp
string s = $@"User {name}
saved at C:\Logs\{name}.txt";
```

Note: `$@"..."` and `@$"..."` both work — the order doesn't matter.

### Raw string literals — `"""..."""` (C# 11+)

For when escape characters get out of hand:

```csharp
string json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;
```

Rules:
- Open and close with **at least three** `"`. Use more if your string contains `"""`.
- The closing `"""` determines the **indentation** — any leading whitespace up to that column is stripped from every line.

```csharp
// All four "lines" are de-indented based on the closing """ position
string regex = """
    (?<area>\d{3})
    -
    (?<num>\d{4})
    """;
// Equivalent to: "(?<area>\\d{3})\n-\n(?<num>\\d{4})"
```

Interpolated raw strings — `$"""..."""`:

```csharp
string name = "Alice";
string json = $$"""
    {
        "name": "{{name}}",
        "version": "{{ VersionString }}"
    }
    """;
```

Notice the `$$` — when you have **two** `$`s, the placeholders use **double braces** `{{ }}` instead of single `{ }`. This lets you embed JSON/CSS/code where single `{ }` are literal.

### UTF-8 string literals — `"..."u8` (C# 11+)

Produces a `ReadOnlySpan<byte>` containing the UTF-8 encoding, no allocation:

```csharp
ReadOnlySpan<byte> bytes = "Hello"u8;        // 5 bytes
ReadOnlySpan<byte> http = "HTTP/1.1 200 OK\r\n"u8;
```

Useful for parsing network protocols, file headers, and anything where you want the byte representation without `Encoding.UTF8.GetBytes`.

---

## String operations

```csharp
string s = "Hello, World!";

// Length and indexing
s.Length;                  // 13
s[0];                      // 'H'
s[^1];                     // '!' — last char (C# 8+ index from end)

// Substring
s.Substring(0, 5);         // "Hello" — start, length
s[..5];                    // "Hello" — range syntax
s[7..];                    // "World!"
s[7..12];                  // "World"

// Search
s.Contains("World");       // true
s.IndexOf('o');            // 4 — first 'o'
s.IndexOf('o', 5);         // 8 — starting from index 5
s.LastIndexOf('o');        // 8
s.StartsWith("Hello");     // true
s.EndsWith("!");           // true

// Case
s.ToUpper();               // "HELLO, WORLD!"
s.ToLower();               // "hello, world!"
s.ToUpperInvariant();      // for machine-readable comparisons (no Turkish I etc.)

// Replace
s.Replace("World", "C#");  // "Hello, C#!"

// Trim
"  spaced  ".Trim();       // "spaced"
"  spaced  ".TrimStart();
"  spaced  ".TrimEnd();
"...hello...".Trim('.');   // trim specific chars

// Split / join
"a,b,c".Split(',');        // ["a", "b", "c"]
string.Join("-", new[] { "a", "b", "c" });   // "a-b-c"
string.Join(", ", names);  // common idiom

// Equality
s == "Hello, World!";      // true — reference type, but == uses value equality for string
s.Equals("HELLO, WORLD!", StringComparison.OrdinalIgnoreCase);  // true

// IsNullOrEmpty / IsNullOrWhiteSpace
string.IsNullOrEmpty(null);      // true
string.IsNullOrEmpty("");        // true
string.IsNullOrEmpty(" ");       // false
string.IsNullOrWhiteSpace(" ");  // true — covers null, empty, and whitespace
```

---

## Strings and equality

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(a == b);          // true
Console.WriteLine(a.Equals(b));     // true
Console.WriteLine(ReferenceEquals(a, b)); // true (interning, see below)

string c = new string(new[] {'h','e','l','l','o'});
Console.WriteLine(a == c);                  // true — value equality
Console.WriteLine(ReferenceEquals(a, c));   // false — different references
```

`==` on strings does **value comparison**. This is one of two exceptions to the rule that `==` on reference types means reference equality (the other being `Nullable<T>`). The C# language hard-codes this for strings.

For comparisons with case or culture sensitivity, use `StringComparison`:

```csharp
"HELLO".Equals("hello", StringComparison.OrdinalIgnoreCase);   // true
string.Equals("café", "cafe", StringComparison.InvariantCultureIgnoreCase); // false — different chars

// Sort-style comparisons
string.Compare("apple", "banana");                  // negative — apple < banana
string.Compare("Apple", "apple", StringComparison.Ordinal);          // negative
string.Compare("Apple", "apple", StringComparison.OrdinalIgnoreCase); // 0
```

`StringComparison` values you should know:
- `Ordinal` — byte-by-byte. Fastest, no culture awareness.
- `OrdinalIgnoreCase` — same, case-insensitive (ASCII rules).
- `InvariantCulture` — culture-aware but using the "invariant" culture. Good for portable formatting.
- `CurrentCulture` — uses the user's culture; only for UI-facing comparisons.

**For machine-readable strings (config keys, JSON properties, etc.), always use `Ordinal` / `OrdinalIgnoreCase`.** Culture-aware comparisons are slower and have surprising rules (Turkish I, German ß, etc.).

---

## String interning

The runtime keeps a pool of unique string literals. Equal literals share the same reference:

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b));   // true — same interned instance

string c = string.Intern("hel" + "lo");     // explicit intern (rare)
Console.WriteLine(ReferenceEquals(a, c));   // true
```

Runtime strings (from `Console.ReadLine`, file I/O, `new string`) are **not** interned unless you call `string.Intern` manually. Don't lean on interning — it's a memory optimization, not a contract.

[Chapter 09 §12](../09-MemoryPerformance/12-StringInterning.md) covers the intern table in detail.

---

## StringBuilder — when to build incrementally

For O(N) construction (rather than O(N²) with `+`):

```csharp
var sb = new StringBuilder();
sb.Append("Hello");
sb.Append(", ");
sb.Append("World");
sb.AppendLine("!");
sb.AppendFormat("Length so far: {0}", sb.Length);
string result = sb.ToString();
```

Internally, `StringBuilder` holds a `char[]` that doubles when full — amortized O(1) per Append.

When to NOT use StringBuilder:
- Building one or two strings: `+` or `$"..."` is clearer.
- Compile-time-known templates: `$"..."` is faster (compiler builds the call directly).
- High-frequency formatting: `string.Create` with a span is often faster.

Rule of thumb: if you're concatenating more than ~5-10 strings in a loop, switch to `StringBuilder`.

---

## Chars and Unicode — the subtle part

A `char` is a **UTF-16 code unit**, not a Unicode codepoint. For codepoints in the Basic Multilingual Plane (BMP, U+0000 to U+FFFF), one char = one codepoint. For codepoints outside (most emoji, some CJK extensions), one codepoint = **two** chars (a surrogate pair).

```csharp
string emoji = "😀";          // U+1F600
Console.WriteLine(emoji.Length);  // 2 — it's a surrogate pair, not 1
Console.WriteLine(emoji[0]);      // high surrogate (printed as ?)
Console.WriteLine(emoji[1]);      // low surrogate
```

To iterate properly by codepoint:

```csharp
foreach (Rune r in emoji.EnumerateRunes()) {
    Console.WriteLine($"U+{r.Value:X4} {r}");
}
// U+1F600 😀
```

Or:
```csharp
var enumerator = StringInfo.GetTextElementEnumerator(text);
while (enumerator.MoveNext()) {
    Console.WriteLine(enumerator.GetTextElement());  // honors grapheme clusters
}
```

**Grapheme clusters** are even higher level — `e + ́` (combining acute) is one user-perceived "character" but two codepoints. For user-facing length, use `StringInfo`. For most code, `Length` (UTF-16 code units) is fine.

---

## Parsing strings to numbers (recap)

```csharp
int.Parse("42");
int.TryParse("42", out int n);
int.TryParse(input, NumberStyles.Integer, CultureInfo.InvariantCulture, out var v);

double.Parse("3.14", CultureInfo.InvariantCulture);
DateTime.Parse("2026-05-19", CultureInfo.InvariantCulture);
DateTime.TryParseExact("2026-05-19", "yyyy-MM-dd", CultureInfo.InvariantCulture, DateTimeStyles.None, out var d);
```

---

## Common string idioms

```csharp
// Null-safe length check
if (string.IsNullOrEmpty(s)) return;
if (string.IsNullOrWhiteSpace(s)) return;

// Default value via ??
string display = userName ?? "(anonymous)";

// Conditional string
string suffix = count == 1 ? "" : "s";
string msg = $"{count} item{suffix}";

// Build a comma-separated list
string list = string.Join(", ", names);
string listOr = string.Join(" or ", lastTwo);

// Pad
"42".PadLeft(5, '0');     // "00042"
"abc".PadRight(10);        // "abc       "

// Format with custom rules
$"{value,10:N2}"   // right-align in 10 cols, 2 decimal places

// Repeat a string (no built-in operator; use:)
new string('-', 40);      // "----------------------------------------"
string.Concat(Enumerable.Repeat("ab", 3));   // "ababab"

// Reverse (no built-in; need extra work for surrogate pairs!)
new string(s.Reverse().ToArray());   // works only for BMP-only strings
```

---

## Strings and Span — high-performance slicing

Since .NET Core 2.1, `string` interoperates with `Span<char>` / `ReadOnlySpan<char>`:

```csharp
string s = "Hello, World!";
ReadOnlySpan<char> span = s;        // implicit conversion
ReadOnlySpan<char> hello = s.AsSpan(0, 5);   // no allocation
hello.SequenceEqual("Hello".AsSpan());

// Most parse methods have Span overloads
int.TryParse("42".AsSpan(), out int n);
```

Use spans on hot paths to avoid `Substring` allocations. Full coverage in [Chapter 09 §05](../09-MemoryPerformance/05-Span.md).

---

## Encoding — when strings become bytes

C# strings are UTF-16 in memory. When reading/writing files or networks, you choose an encoding:

```csharp
byte[] bytes = Encoding.UTF8.GetBytes("Hello");      // 5 bytes: 72,101,108,108,111
string back = Encoding.UTF8.GetString(bytes);

byte[] u16 = Encoding.Unicode.GetBytes("Hello");      // 10 bytes (UTF-16)
byte[] latin1 = Encoding.Latin1.GetBytes("Hello");    // 5 bytes (ISO-8859-1)
```

**Always specify the encoding when reading/writing files**. `File.ReadAllText` defaults to UTF-8 (with BOM detection) since .NET 6, but be explicit:

```csharp
File.ReadAllText("data.txt", Encoding.UTF8);
File.WriteAllText("out.txt", text, new UTF8Encoding(encoderShouldEmitUTF8Identifier: false)); // no BOM
```

---

## Common bugs

- **Building strings with `+` in a loop** — quadratic time. Use `StringBuilder`.
- **`string s = null; s.Length;`** — `NullReferenceException`. Check first or use `s?.Length` (returns `int?`).
- **`s.Equals("HELLO")` for case-insensitive** — defaults to case-sensitive. Pass `StringComparison.OrdinalIgnoreCase`.
- **`s.Length` for emoji** — counts UTF-16 code units, not user-perceived characters. Use `StringInfo` if you need grapheme count.
- **`string.Compare("Apple", "apple")` for sorting** — uses current culture by default. Pass `StringComparison.Ordinal` for machine ordering.
- **`int.Parse(userInput)` without try** — throws on bad input. Use `int.TryParse`.
- **`"hello" == new string(...)`** — `==` on strings DOES compare by value (one of the few exceptions for reference types). Some tutorials get this wrong.
- **`"".Split(',')`** — returns `[""]`, not `[]`. Filter with `where !string.IsNullOrEmpty(...)` if you want empty entries skipped, or use `Split(',', StringSplitOptions.RemoveEmptyEntries)`.

---

## Performance summary

| Operation | Cost |
|---|---|
| `s.Length` | O(1) — cached |
| `s[i]` | O(1) |
| `s + t` (single) | O(n + m) — one allocation |
| `s + t1 + t2 + ... + tn` (loop) | O(n²) — many allocations |
| `string.Concat(...)` | O(total len) — one allocation |
| `string.Join(sep, items)` | O(total len) — one allocation |
| `$"{x} {y}"` | optimized by compiler to one call |
| `StringBuilder.Append` in loop | amortized O(1) |
| `s.Substring(start, len)` | O(len) — allocates a copy |
| `s.AsSpan(start, len)` | O(1) — no allocation |
| `s == t` | O(n) — character by character |

---

## When to use what

| Goal | Use |
|---|---|
| Static text | `"..."` |
| File path | `@"..."` |
| Embedded expressions | `$"..."` |
| Multi-line embedded content (JSON, SQL, regex) | `"""..."""` |
| Bytes of a small literal | `"..."u8` |
| Building incrementally | `StringBuilder` |
| Slicing without allocating | `s.AsSpan(...)` |
| Comparing for equality | `==` or `Equals(...)` |
| Comparing case-insensitive | `Equals(..., StringComparison.OrdinalIgnoreCase)` |
| Comparing for sort order | `string.Compare(...)` with explicit `StringComparison` |
| Converting to bytes | `Encoding.UTF8.GetBytes(...)` |

→ Next: [04-Operators.md](04-Operators.md)
