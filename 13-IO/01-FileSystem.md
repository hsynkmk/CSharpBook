# File System — File, Directory, Path

## What it is

The `System.IO` namespace exposes the file system through static helper classes and instance info classes:

- **`File`** — static methods for file operations (read, write, copy, delete, exists).
- **`Directory`** — static methods for directories (create, enumerate, delete).
- **`Path`** — pure string manipulation of paths (combine, get extension, get directory). No I/O.
- **`FileInfo` / `DirectoryInfo`** — instance wrappers that cache metadata; efficient for repeated queries on the same path.

```csharp
File.WriteAllText("config.json", json);
string content = File.ReadAllText("config.json");
bool exists = File.Exists("config.json");

Directory.CreateDirectory("logs/2026");
foreach (var f in Directory.EnumerateFiles("logs", "*.txt", SearchOption.AllDirectories))
    Console.WriteLine(f);

string full = Path.Combine("logs", "2026", "app.log");
string ext = Path.GetExtension(full);   // ".log"
```

---

## `File` — common operations

```csharp
// Whole-file (convenient, loads everything into memory)
string text = File.ReadAllText(path);
string[] lines = File.ReadAllLines(path);
byte[] bytes = File.ReadAllBytes(path);

File.WriteAllText(path, "content");        // overwrites
File.AppendAllText(path, "more\n");        // appends
File.WriteAllLines(path, new[] { "a", "b" });

// Streaming (lazy, low memory)
foreach (string line in File.ReadLines(path))   // IEnumerable — one line at a time
    Process(line);

// Metadata operations
File.Copy(src, dst, overwrite: true);
File.Move(src, dst, overwrite: true);       // overwrite param added in .NET Core 3.0
File.Delete(path);                           // no error if missing? NO — throws if dir missing
bool ok = File.Exists(path);
```

### `ReadAllLines` vs `ReadLines`

- `ReadAllLines` — reads the entire file into a `string[]`. Eager, all in memory.
- `ReadLines` — returns `IEnumerable<string>`, reads lazily. Use for large files to avoid loading everything.

```csharp
// Memory-friendly: process a 10-GB log without loading it all
long errorCount = File.ReadLines("huge.log").Count(l => l.Contains("ERROR"));
```

---

## Async file I/O

```csharp
string text = await File.ReadAllTextAsync(path, cancellationToken);
await File.WriteAllTextAsync(path, json, cancellationToken);
byte[] data = await File.ReadAllBytesAsync(path);

await foreach (var line in ReadLinesAsync(path)) { ... }   // .NET 7+: File.ReadLinesAsync
```

Async file methods free the calling thread during disk I/O. For server apps (ASP.NET), prefer async to avoid thread-pool starvation. For CLI tools doing one read, sync is simpler and fine.

> Note: on many OSes file I/O isn't truly async at the kernel level for regular files; the runtime may use thread-pool offloading. The benefit is still real for scalability.

---

## `Directory`

```csharp
Directory.CreateDirectory(path);    // idempotent — no error if exists, creates intermediates
Directory.Delete(path, recursive: true);
bool exists = Directory.Exists(path);

// Enumeration — prefer Enumerate* (lazy) over Get* (eager array)
foreach (var dir in Directory.EnumerateDirectories(root))
    Console.WriteLine(dir);

foreach (var file in Directory.EnumerateFiles(root, "*.cs", SearchOption.AllDirectories))
    Console.WriteLine(file);

string cwd = Directory.GetCurrentDirectory();
string temp = Path.GetTempPath();
```

`EnumerateFiles` returns `IEnumerable<string>` (lazy, starts yielding immediately). `GetFiles` returns `string[]` (eager, blocks until full directory scanned). Prefer `Enumerate*` for large directories.

### Search options

- `SearchOption.TopDirectoryOnly` (default) — only the immediate folder.
- `SearchOption.AllDirectories` — recursive.
- `EnumerationOptions` (newer overload) — fine-grained control: `RecurseSubdirectories`, `IgnoreInaccessible`, `MaxRecursionDepth`, `AttributesToSkip`.

```csharp
var opts = new EnumerationOptions {
    RecurseSubdirectories = true,
    IgnoreInaccessible = true,        // skip permission-denied folders instead of throwing
    MaxRecursionDepth = 5
};
foreach (var f in Directory.EnumerateFiles(root, "*.log", opts)) { ... }
```

---

## `Path` — pure string operations

`Path` never touches the disk. It manipulates path strings according to platform rules.

```csharp
Path.Combine("a", "b", "c.txt");        // "a/b/c.txt" (or a\b\c.txt on Windows)
Path.GetFileName("a/b/c.txt");           // "c.txt"
Path.GetFileNameWithoutExtension(...);   // "c"
Path.GetExtension("c.txt");              // ".txt"
Path.GetDirectoryName("a/b/c.txt");      // "a/b"
Path.GetFullPath("../x");                // resolves to absolute
Path.IsPathRooted("/etc");               // true
Path.GetTempFileName();                  // creates a 0-byte temp file, returns path
Path.GetRandomFileName();                // random name, no file created
Path.ChangeExtension("a.txt", ".bak");   // "a.bak"

// Span-based (allocation-free) variants exist:
ReadOnlySpan<char> ext = Path.GetExtension("file.txt".AsSpan());
```

### Always use `Path.Combine`, never string concatenation

```csharp
// ✗ — breaks on Windows/Linux differences, double-slashes
string bad = dir + "/" + file;

// ✓ — handles separators correctly per OS
string good = Path.Combine(dir, file);
```

`Path.DirectorySeparatorChar` is `\` on Windows, `/` on Unix. `Path.Combine` uses the right one.

---

## `FileInfo` / `DirectoryInfo`

When you query the **same path multiple times**, `FileInfo` caches metadata after the first access — fewer syscalls than repeated `File.GetX` calls.

```csharp
var fi = new FileInfo(path);
if (fi.Exists) {
    Console.WriteLine($"{fi.Length} bytes, modified {fi.LastWriteTimeUtc}");
    fi.CopyTo(backup, overwrite: true);
}

var di = new DirectoryInfo(root);
foreach (var sub in di.GetDirectories()) { ... }
long totalSize = di.EnumerateFiles("*", SearchOption.AllDirectories).Sum(f => f.Length);
```

Call `fi.Refresh()` to re-read cached metadata if the file may have changed.

Rule of thumb:
- One-off operation → static `File`/`Directory` methods.
- Repeated metadata queries on the same path → `FileInfo`/`DirectoryInfo`.

---

## Atomic file writes (the temp + rename pattern)

A naive `File.WriteAllText` can leave a **half-written file** if the process crashes mid-write. For config/data files, write to a temp file then atomically rename:

```csharp
public static void WriteAtomic(string path, string content) {
    string tmp = path + ".tmp." + Path.GetRandomFileName();
    File.WriteAllText(tmp, content);
    File.Move(tmp, path, overwrite: true);   // rename is atomic on the same volume
}
```

`File.Move` (rename) within the same volume is atomic on most file systems — readers see either the old file or the complete new file, never a partial one. This is how databases and config writers avoid corruption.

For durability (survive power loss), also `FileStream.Flush(flushToDisk: true)` before the rename.

---

## Common patterns

### Ensure a directory exists before writing

```csharp
Directory.CreateDirectory(Path.GetDirectoryName(filePath)!);   // idempotent
File.WriteAllText(filePath, content);
```

`CreateDirectory` is a no-op if the directory already exists — no need to check first.

### Safe delete

```csharp
if (File.Exists(path)) File.Delete(path);
// or just:
try { File.Delete(path); } catch (FileNotFoundException) { }   // Delete on missing file doesn't throw, but missing DIR does
```

`File.Delete` on a missing **file** is silent; on a missing **directory** in the path it throws `DirectoryNotFoundException`.

### Recursive directory size

```csharp
long DirSize(string path) =>
    new DirectoryInfo(path)
        .EnumerateFiles("*", SearchOption.AllDirectories)
        .Sum(f => f.Length);
```

---

## Common bugs

### Path traversal vulnerability

```csharp
// ⚠ — user input "../../etc/passwd" escapes the intended folder
string path = Path.Combine(baseDir, userInput);

// ✓ — verify the resolved path stays inside baseDir
string full = Path.GetFullPath(Path.Combine(baseDir, userInput));
if (!full.StartsWith(Path.GetFullPath(baseDir) + Path.DirectorySeparatorChar))
    throw new UnauthorizedAccessException();
```

Always validate user-supplied path components against directory traversal.

### Forgetting overwrite throws

```csharp
File.Copy(src, dst);   // ⚠ — throws IOException if dst exists
File.Copy(src, dst, overwrite: true);   // ✓
```

### Case sensitivity

File systems differ: Windows/macOS default case-**insensitive**, Linux case-**sensitive**. `"File.txt"` and `"file.txt"` may or may not be the same file. Don't assume.

### Locked files

```csharp
// Another process has the file open exclusively
File.ReadAllText(path);   // ⚠ — IOException: file in use
```

Use `FileShare` flags via `FileStream` when concurrent access is expected (see [02-Streams.md](02-Streams.md)).

### Long paths on Windows

Paths > 260 chars historically threw `PathTooLongException`. Modern .NET + Windows 10+ supports long paths if enabled (manifest + registry / `\\?\` prefix). Don't rely on short paths.

---

## Performance notes

- `ReadLines`/`EnumerateFiles` (lazy) beat `ReadAllLines`/`GetFiles` (eager) for large inputs — lower memory, faster time-to-first-result.
- For high-throughput I/O, use `FileStream` with a buffer or `RandomAccess` (`System.IO.RandomAccess`, .NET 6+) for offset-based reads without a stream position.
- `File.ReadAllBytesAsync` is fine for moderate files; for huge files, stream in chunks.
- `FileInfo` caching avoids repeated `stat` syscalls.

---

## When to use what

| Need | Use |
|---|---|
| Read/write a small whole file | `File.ReadAllText` / `WriteAllText` |
| Process a large file line by line | `File.ReadLines` (lazy) |
| Binary, large, or seeking | `FileStream` (see §02) |
| Repeated metadata queries | `FileInfo` / `DirectoryInfo` |
| Build/inspect a path string | `Path` (never string concat) |
| Crash-safe writes | temp + `File.Move` (atomic rename) |
| Offset reads without a stream | `RandomAccess.Read` (.NET 6+) |

---

## Summary

- `File` / `Directory` — static I/O helpers. `Path` — pure string ops, no disk.
- Prefer lazy `ReadLines` / `EnumerateFiles` for large inputs.
- Use async methods in server code to avoid blocking thread-pool threads.
- Always `Path.Combine`, never concatenate path strings.
- Use temp + atomic rename for crash-safe writes.
- Validate user-supplied paths against traversal; account for case sensitivity and locking.

→ Next: [02-Streams.md](02-Streams.md)
