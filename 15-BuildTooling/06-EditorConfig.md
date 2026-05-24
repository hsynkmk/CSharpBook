# EditorConfig

## What it is

`.editorconfig` is a plain-text file that defines **coding style and formatting rules** for a codebase, honored by editors, the compiler's style analyzers, and `dotnet format`. It makes style consistent across the whole team regardless of IDE — and can enforce it at build time.

```ini
# .editorconfig at repo root
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = lf
insert_final_newline = true
charset = utf-8
```

It's an industry standard (not .NET-specific) extended by .NET with C#-specific rules (naming, `var` usage, expression-body preferences, analyzer severities).

---

## File discovery and hierarchy

EditorConfig files apply hierarchically. The editor/compiler walks **up** from a file's directory, applying all `.editorconfig` files until one with `root = true`.

```
repo/
├── .editorconfig            (root = true)   ← base rules
├── src/
│   └── .editorconfig                          ← overrides/adds for src
└── tests/
    └── .editorconfig                          ← relaxed rules for tests
```

Closer files override farther ones. `root = true` stops the upward search. This lets you relax rules for tests/generated code while keeping production code strict.

---

## Section globs

```ini
[*]                  # all files
[*.cs]               # C# files
[*.{cs,vb}]          # C# and VB
[*.{json,xml,yml}]   # data files
[**/Migrations/*.cs] # generated migrations
[tests/**.cs]        # test files
```

---

## Universal formatting rules

```ini
[*]
indent_style = space
indent_size = 4
tab_width = 4
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
charset = utf-8

[*.{yml,yaml,json}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false   # trailing spaces are line breaks in markdown
```

---

## C# language style rules

.NET-specific rules controlling code style. Each has an option value and an optional severity.

```ini
[*.cs]
# 'var' usage
csharp_style_var_for_built_in_types = true:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = false:suggestion

# Expression-bodied members
csharp_style_expression_bodied_methods = when_on_single_line:suggestion
csharp_style_expression_bodied_properties = true:suggestion

# Pattern matching
csharp_style_pattern_matching_over_is_with_cast_check = true:warning
csharp_style_prefer_switch_expression = true:suggestion

# Null checking
csharp_style_throw_expression = true:suggestion
csharp_style_conditional_delegate_call = true:suggestion   # obj?.Invoke()

# Modern features
csharp_style_prefer_primary_constructors = true:suggestion
csharp_prefer_braces = true:warning
csharp_style_namespace_declarations = file_scoped:warning   # file-scoped namespaces

# 'this.' qualification
dotnet_style_qualification_for_field = false:suggestion
dotnet_style_qualification_for_property = false:suggestion

# Readonly / modifiers
dotnet_style_readonly_field = true:suggestion
csharp_preferred_modifier_order = public,private,protected,internal,static,readonly,async:suggestion

# using directives
csharp_using_directive_placement = outside_namespace:warning
dotnet_sort_system_directives_first = true
```

### The `value:severity` syntax

```ini
csharp_prefer_braces = true:warning
```

- **value** (`true`/`false` or an option) — the preferred style.
- **severity** (`error`/`warning`/`suggestion`/`silent`/`none`) — how strongly to enforce.

A `warning` shows a build warning (and an error if `EnforceCodeStyleInBuild` + warnings-as-errors). A `suggestion` shows an IDE hint only.

---

## Naming conventions

The most powerful (and verbose) part. Three pieces: **symbols** (what), **style** (how it should look), **rule** (tie them together with severity).

```ini
[*.cs]
# 1. Define the symbol group: private fields
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

# 2. Define the style: _camelCase
dotnet_naming_style.underscore_camel.capitalization = camel_case
dotnet_naming_style.underscore_camel.required_prefix = _

# 3. The rule: private fields must use _camelCase, as a warning
dotnet_naming_rule.private_fields_underscore.symbols = private_fields
dotnet_naming_rule.private_fields_underscore.style = underscore_camel
dotnet_naming_rule.private_fields_underscore.severity = warning

# Constants in PascalCase
dotnet_naming_symbols.constants.applicable_kinds = field
dotnet_naming_symbols.constants.required_modifiers = const
dotnet_naming_style.pascal.capitalization = pascal_case
dotnet_naming_rule.constants_pascal.symbols = constants
dotnet_naming_rule.constants_pascal.style = pascal
dotnet_naming_rule.constants_pascal.severity = warning

# Interfaces must start with I
dotnet_naming_symbols.interfaces.applicable_kinds = interface
dotnet_naming_style.i_prefix.capitalization = pascal_case
dotnet_naming_style.i_prefix.required_prefix = I
dotnet_naming_rule.interfaces_i.symbols = interfaces
dotnet_naming_rule.interfaces_i.style = i_prefix
dotnet_naming_rule.interfaces_i.severity = warning
```

See [Chapter 17 §01](../17-BestPractices/01-NamingConventions.md) for the conventions these encode.

---

## Configuring analyzer severities

`.editorconfig` is also where you set the severity of `CAxxxx`/`IDExxxx`/custom analyzer rules (see [05-RoslynAnalyzers.md](05-RoslynAnalyzers.md)):

```ini
[*.cs]
dotnet_diagnostic.CA2007.severity = error    # ConfigureAwait
dotnet_diagnostic.CA1822.severity = suggestion
dotnet_diagnostic.IDE0090.severity = warning

# Relax rules for tests
[tests/**.cs]
dotnet_diagnostic.CA1707.severity = none      # underscores in names (test method naming)
dotnet_diagnostic.CA2007.severity = none
```

This unifies *style preferences* and *analyzer enforcement* in one file per scope.

---

## Enforcing at build time

```xml
<PropertyGroup>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>   <!-- run IDExxxx style rules during build -->
</PropertyGroup>
```

Without this, `IDExxxx` style rules only show in the IDE. With it, they fire during `dotnet build` (and CI). Combined with `dotnet format --verify-no-changes`, you can gate PRs on formatting/style.

```bash
dotnet format --verify-no-changes    # CI: exit non-zero if not formatted to .editorconfig
```

---

## Generating a starting `.editorconfig`

```bash
dotnet new editorconfig    # creates a comprehensive default .editorconfig
```

Visual Studio can also generate one from current settings, and dotnet/roslyn publish a well-commented reference. Start from a generated file and trim/adjust.

---

## Common bugs / gotchas

### Forgetting `root = true`

Without it, the search continues up past the repo, possibly picking up unrelated `.editorconfig` files (e.g., in your home directory). Set `root = true` at the repo root.

### Style rules in IDE but not build

`IDExxxx` rules need `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>` to fire at build. Otherwise they're IDE-only suggestions.

### Naming rules silently not applying

Naming convention rules are order- and completeness-sensitive — all three parts (symbol, style, rule) must be present and consistent. A typo in a key name makes the rule silently inert. Test by introducing a violation.

### Conflicting rules across files

A child `.editorconfig` overriding a parent can cause confusion. Keep the hierarchy shallow and document overrides (e.g., why tests relax a rule).

---

## Summary

- `.editorconfig` defines formatting, C# style, naming conventions, and analyzer severities — honored across editors and the build.
- Rules apply hierarchically up to `root = true`; closer files override.
- Style rules use `value:severity`; naming needs symbol + style + rule triplets.
- Configure `CAxxxx`/`IDExxxx` severities here too; relax rules per-scope (e.g., tests).
- Enable `<EnforceCodeStyleInBuild>` to enforce style at build; gate CI with `dotnet format --verify-no-changes`.
- Start from `dotnet new editorconfig`; always set `root = true`.

→ Next: [07-Debugging.md](07-Debugging.md)
