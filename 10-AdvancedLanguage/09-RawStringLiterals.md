# Raw String Literals (C# 11+)

## What it is

A **raw string literal** uses `"""` (triple-quote) delimiters and lets you write strings with embedded quotes, backslashes, and newlines — no escaping needed.

```csharp
string regex = """\d{3}-\d{4}""";    // no need for \\d
string json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;
```

Added in C# 11. Replaces awkward escape sequences for embedded code (JSON, regex, SQL, XML).

---

## Why it exists

Pre-C# 11 string options:

```csharp
// Regular: escape everything
string a = "\"hello\"\nworld";

// Verbatim (@): no escapes, but quotes still need ""
string b = @"""hello""
world";

// Both fail when string contains MANY quotes / special chars
string json = "{\"name\":\"Alice\",\"data\":[\"a\",\"b\"]}";   // ugly
string json2 = @"{""name"":""Alice"",""data"":[""a"",""b""]}"; // less ugly
```

Raw strings just sidestep escaping:

```csharp
string json = """{"name":"Alice","data":["a","b"]}""";
```

Embedded JSON, regex, SQL, XML, HTML — all become readable.

---

## Basic syntax

```csharp
string s = """raw content""";
```

Open and close with at least **three** `"`. Inside, any characters are taken literally (no escape sequences interpreted).

To include `"""` inside the string, use **more** opening/closing quotes:

```csharp
string s = """"contains """ inside"""";   // 4 quotes outside
```

The rule: use one MORE quote outside than the longest run inside.

---

## Multi-line raw strings

When you span multiple lines, the closing `"""` determines the indentation. Leading whitespace matching the closing indent is stripped from each line:

```csharp
string s = """
    First line.
    Second line.
    """;
// s is "First line.\nSecond line." — the 4-space indent before each line is stripped
```

The closing `"""` MUST be on its own line (multi-line form requires this). Its column position is the "indent baseline."

```csharp
string s = """
        Indented one extra level
    Standard indent
        """;
// s is "    Indented one extra level\nStandard indent\n" — relative to the closing """
```

This makes embedded code blocks read naturally with your surrounding indentation.

---

## Interpolated raw strings

Combine `$` with `"""`:

```csharp
string name = "Alice";
string s = $"""Hello, {name}!""";
```

Interpolation works inside raw strings.

Subtlety: if your string contains literal `{` or `}`, you'd have to escape them in regular interpolation. Raw strings have an alternative — use **more `$` symbols** to require more `{`s:

```csharp
string name = "Alice";
string s = $"""Use {name} here""";       // {name} is interpolation
string s2 = $$"""Use {name} here""";    // {name} is LITERAL; need {{name}} for interpolation
string s3 = $$"""Hi, {{name}}!""";       // {{name}} is interpolation; { and } are literal
```

Each `$` adds one to the brace requirement. Useful for embedding JSON, where `{` `}` are common literals.

---

## Embedded code examples

### JSON template

```csharp
string id = "42";
string name = "Alice";
string json = $$"""
    {
        "id": {{id}},
        "name": "{{name}}",
        "metadata": {
            "tag": "user"
        }
    }
    """;
```

The `{{id}}` is interpolation. The literal `{` `}` for JSON object syntax are unescaped.

### Regex

```csharp
string pattern = """\d{3}-\d{4}""";
```

No backslash doubling. Cleaner than `@"\d{3}-\d{4}"` only slightly, but with embedded quotes it shines:

```csharp
string pattern = """([""'])(?:(?!\1).)*\1""";   // matches "quoted" or 'quoted'
```

vs the escape-soup equivalent.

### SQL

```csharp
string sql = """
    SELECT *
    FROM Users
    WHERE Name LIKE 'A%'
    ORDER BY CreatedAt DESC
    """;
```

Multi-line SQL with single quotes; no escaping.

### XML / HTML

```csharp
string html = """
    <div class="container">
        <h1>Hello</h1>
        <p>Welcome</p>
    </div>
    """;
```

---

## Single-line raw

```csharp
string s = """hello""";    // single-line raw, equivalent to "hello"
```

For single-line use without quotes/backslashes, no real benefit over `"..."`. The win is for content with quotes:

```csharp
string greeting = """The user said "hi"""";
```

Three opening, four closing — wait, that's wrong. Let me re-do:

```csharp
string greeting = """The user said "hi" gracefully.""";
```

Three quotes open and close. The internal `"` doesn't conflict because the string is opened/closed by `"""`.

If the string ITSELF contains `"""`:

```csharp
string s = """"This has """ inside"""";   // 4-quote delimiters
```

Open and close with 4. The internal 3-quote run is unambiguously literal.

---

## Indentation rules — the details

For multi-line raw strings:

1. The closing `"""` is on its own line.
2. Its column determines the "baseline indent."
3. Each line of the content must have at least the baseline indent (in spaces or tabs).
4. The baseline indent is **stripped** from each line.

```csharp
string s = """
    Line 1
    Line 2
    """;
// Baseline: 4 spaces (column of """)
// Line 1: "    Line 1" → "Line 1" after strip
// Result: "Line 1\nLine 2"
```

If a line has LESS indent than the baseline, compile error:

```csharp
string s = """
    Line 1
 Less indented   // ⚠ — less than baseline
    """;
```

This catches sloppy alignment.

For extra indentation (relative to baseline):

```csharp
string s = """
    Line 1
        Indented further
    Line 3
    """;
// Result: "Line 1\n    Indented further\nLine 3"
// The 4-space relative indent is preserved.
```

The baseline is stripped; anything additional is kept.

---

## Leading / trailing newlines

The first newline after the opening `"""` is NOT included in the string. The last newline before the closing `"""` is NOT included.

```csharp
string s = """
    Hello
    """;
// s is "Hello" — no leading/trailing \n
```

So multi-line raws don't add surprise blank lines.

If you want a trailing newline:

```csharp
string s = """
    Hello

    """;   // Note the blank line before """
// s is "Hello\n"
```

The blank line becomes a literal `\n`.

---

## Internals — compile-time string

A raw string is just a string at runtime. The runtime form is identical to:

```csharp
string s = "regular form";
```

The compiler processes the raw syntax (indentation stripping, escape disabling) at compile time. Output: a regular `System.String` literal in the assembly.

For interpolated raw strings, the compiler emits the same `string.Concat` / `InterpolatedStringHandler` calls as regular interpolated strings.

No runtime cost. Pure compile-time convenience.

---

## When to use

✓ Embedded code (JSON, regex, SQL, XML, HTML).
✓ Multi-line strings with no escape mess.
✓ Strings with many quotes or backslashes.
✓ Tests with literal expected output.

✗ Single short strings — `"hello"` is fine.
✗ Strings with simple escapes — `\n` and `\"` are still OK to use.

---

## Common patterns

### Test fixtures

```csharp
[Fact]
public void Parse_ValidJson_ReturnsExpected() {
    string input = """
        {
            "name": "Alice",
            "age": 30
        }
        """;
    var result = JsonSerializer.Deserialize<Person>(input);
    result!.Name.Should().Be("Alice");
}
```

Embedded JSON in a test, readable.

### Configuration templates

```csharp
string yamlTemplate = $$"""
    version: '3'
    services:
      web:
        image: {{imageName}}
        ports:
          - "{{port}}:80"
        environment:
          ASPNETCORE_ENVIRONMENT: {{env}}
    """;
```

YAML / TOML / INI templates with interpolation.

### Code generation

```csharp
public string GenerateClass(string name, IEnumerable<string> props) {
    var propsCode = string.Join("\n    ", props.Select(p => $"public string {p} {{ get; init; }}"));
    return $$"""
        public class {{name}} {
            {{propsCode}}
        }
        """;
}
```

Source generators love this — multi-line code emission without escape soup.

---

## Common bugs

### Wrong baseline indent

```csharp
string s = """
    Hello
  World   // ⚠ — less indent than baseline
    """;
```

Compile error: line is less indented than the closing `"""`. Fix the alignment.

### Forgetting `$$` for JSON-with-interpolation

```csharp
string id = "42";
string json = $"""
    {"id": {id}}
    """;
// ⚠ — { and } are problematic; this might fail or produce wrong output
```

Use double `$$`:

```csharp
string json = $$"""
    {"id": {{id}}}
    """;
```

### Mistaking `"""` count

```csharp
string s = """contains """""";   // 3 opening, 5 internal closing — ambiguous to compiler
```

Use more quotes outside than the longest run inside:

```csharp
string s = """"""contains """""""""";   // hypothetical worst case
```

In practice: 3 or 4 quotes covers nearly everything.

### Trailing whitespace on indentation lines

If the closing `"""` is preceded by tabs/spaces, those must match the content's indent. Mixing tabs and spaces breaks the baseline.

EditorConfig / formatter settings help avoid this.

---

## Performance

Zero runtime cost. Raw strings compile to the same `string` constants as regular strings.

---

## Summary

- `"""..."""` — raw string literal. No escapes interpreted.
- Multi-line: closing `"""` determines indent baseline; that much whitespace stripped from each line.
- `$"""..."""` — interpolated raw string.
- `$$"""...{{x}}..."""` — interpolated raw with `{{ }}` placeholders (literal `{` and `}` allowed).
- Use for embedded code, multi-line content, anything with many quotes.
- Compile-time syntax only; runtime is a regular string.

→ Next: [10-UTF8StringLiterals.md](10-UTF8StringLiterals.md)
