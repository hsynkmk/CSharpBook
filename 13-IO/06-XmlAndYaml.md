# XML and YAML

## XML in .NET — the options

.NET offers several XML APIs at different levels:

| API | Style | Use for |
|---|---|---|
| `XDocument` / LINQ to XML | In-memory tree, LINQ queries | Most XML work today |
| `XmlDocument` | Old W3C DOM | Legacy code |
| `XmlReader` / `XmlWriter` | Forward-only streaming | Huge XML, low memory |
| `XmlSerializer` | Object ↔ XML | POCO mapping |
| `DataContractSerializer` | WCF-style serialization | Legacy WCF / specific control |

For new code, prefer **`XDocument`** (LINQ to XML) for manipulation and **`XmlReader`** for large streaming.

---

## LINQ to XML (`XDocument`)

```csharp
using System.Xml.Linq;

// Parse
XDocument doc = XDocument.Parse(xmlString);
XDocument fromFile = XDocument.Load("data.xml");

// Query with LINQ
var names = doc.Root!
    .Elements("person")
    .Where(p => (int)p.Attribute("age")! >= 18)
    .Select(p => (string)p.Element("name")!)
    .ToList();

// Build
var newDoc = new XDocument(
    new XElement("library",
        new XElement("book",
            new XAttribute("isbn", "123"),
            new XElement("title", "C# in Depth"),
            new XElement("year", 2025))));
newDoc.Save("library.xml");
```

The casts `(int)`, `(string)` on `XElement`/`XAttribute` are convenient conversions. `XDocument` is far more pleasant than the old `XmlDocument` DOM.

### Namespaces

```csharp
XNamespace ns = "http://example.com/schema";
var elements = doc.Descendants(ns + "item");

var el = new XElement(ns + "root", new XElement(ns + "child", "value"));
```

`XNamespace + "localName"` produces a namespace-qualified `XName`. Forgetting namespaces is the #1 reason LINQ-to-XML queries return nothing.

---

## `XmlReader` / `XmlWriter` — streaming

For multi-gigabyte XML, don't load a DOM. Stream forward-only:

```csharp
using var reader = XmlReader.Create("huge.xml", new XmlReaderSettings { IgnoreWhitespace = true });
while (reader.Read()) {
    if (reader.NodeType == XmlNodeType.Element && reader.Name == "record") {
        string id = reader.GetAttribute("id")!;
        // Optionally read this subtree into an XElement for convenience:
        var element = (XElement)XNode.ReadFrom(reader);
        Process(element);
    }
}
```

`XmlReader` reads one node at a time — constant memory regardless of file size. Combine with `XNode.ReadFrom` to materialize just the subtrees you care about.

```csharp
using var writer = XmlWriter.Create("out.xml", new XmlWriterSettings { Indent = true });
writer.WriteStartElement("root");
writer.WriteElementString("name", "Alice");
writer.WriteEndElement();
```

---

## `XmlSerializer` — object mapping

Maps POCOs to/from XML, controlled by attributes:

```csharp
[XmlRoot("person")]
public class Person {
    [XmlElement("full-name")] public string Name { get; set; } = "";
    [XmlAttribute("age")] public int Age { get; set; }
    [XmlArray("hobbies")]
    [XmlArrayItem("hobby")]
    public List<string> Hobbies { get; set; } = new();
    [XmlIgnore] public string Internal { get; set; } = "";
}

var serializer = new XmlSerializer(typeof(Person));

using var sw = new StringWriter();
serializer.Serialize(sw, person);
string xml = sw.ToString();

using var sr = new StringReader(xml);
var p = (Person)serializer.Deserialize(sr)!;
```

### Cache the serializer

```csharp
// ⚠ — XmlSerializer generates and compiles an assembly per type on first construction.
//      Creating it repeatedly leaks dynamic assemblies + is slow.
private static readonly XmlSerializer Serializer = new(typeof(Person));
```

`new XmlSerializer(type)` (the simple constructor) caches generated assemblies internally. But the constructors that take extra args (e.g., `XmlAttributeOverrides`) do **not** cache — they leak. Always cache your serializer instances.

Requirements: `XmlSerializer` needs a public parameterless constructor and serializes public read/write members only.

---

## `DataContractSerializer`

WCF-era serializer. More control over what's serialized via opt-in attributes:

```csharp
[DataContract]
public class Order {
    [DataMember] public int Id { get; set; }
    [DataMember(Name = "customer")] public string Customer { get; set; } = "";
    // Non-[DataMember] fields are NOT serialized (opt-in model)
}

var dcs = new DataContractSerializer(typeof(Order));
using var ms = new MemoryStream();
dcs.WriteObject(ms, order);
ms.Position = 0;
var back = (Order)dcs.ReadObject(ms)!;
```

`DataContractSerializer` is **opt-in** (only `[DataMember]` members), while `XmlSerializer` is **opt-out** (all public members unless `[XmlIgnore]`). Use `DataContractSerializer` for legacy WCF interop; otherwise prefer `XmlSerializer` or JSON.

---

## XML security — XXE

XML parsers can be tricked into reading local files or making network requests via external entities (XXE attack):

```csharp
// ✓ — modern .NET disables DTD processing by default, but be explicit for untrusted input
var settings = new XmlReaderSettings {
    DtdProcessing = DtdProcessing.Prohibit,   // block DTDs / external entities
    XmlResolver = null                          // don't resolve external resources
};
using var reader = XmlReader.Create(stream, settings);
```

Modern .NET defaults are safe (`DtdProcessing.Prohibit`), but always verify when parsing untrusted XML. Newtonsoft and older `XmlDocument` settings differed.

---

## YAML — not built in

.NET has **no built-in YAML support**. The de-facto library is **YamlDotNet** (NuGet).

```csharp
// dotnet add package YamlDotNet
using YamlDotNet.Serialization;

var deserializer = new DeserializerBuilder()
    .WithNamingConvention(CamelCaseNamingConvention.Instance)
    .Build();
var config = deserializer.Deserialize<AppConfig>(yamlText);

var serializer = new SerializerBuilder()
    .WithNamingConvention(CamelCaseNamingConvention.Instance)
    .Build();
string yaml = serializer.Serialize(config);
```

YAML is common for config (Kubernetes, CI pipelines, app settings). For .NET app configuration, though, JSON (`appsettings.json`) is the native choice via `Microsoft.Extensions.Configuration`.

---

## Choosing a format

| Format | Pros | Cons | Use for |
|---|---|---|---|
| JSON | Compact, fast, native STJ, web-standard | No comments, less expressive | APIs, config, most data |
| XML | Schema (XSD), namespaces, comments, attributes | Verbose, slower | Legacy, document formats, SOAP |
| YAML | Human-friendly, comments, anchors | Whitespace-sensitive, no built-in support | Config files (k8s, CI) |

For new APIs and data interchange, **JSON** is the default. XML for interop with existing XML systems. YAML for human-edited config.

---

## Common bugs

### Missing namespace in LINQ to XML

```csharp
doc.Descendants("item")          // ⚠ — returns nothing if items are in a namespace
doc.Descendants(ns + "item")     // ✓
```

### Not caching XmlSerializer

Covered above — repeated `new XmlSerializer(type, overrides)` leaks assemblies.

### XmlSerializer can't serialize interfaces / Dictionary

`XmlSerializer` chokes on interfaces, `Dictionary<,>`, and types without parameterless ctors. Use concrete types and lists, or switch to JSON / `DataContractSerializer`.

### Forgetting to rewind the stream

After `WriteObject(ms, ...)`, set `ms.Position = 0` before `ReadObject`.

---

## Performance notes

- LINQ to XML loads the full tree — fine for moderate documents, not for huge ones.
- `XmlReader` for large/streaming XML — constant memory.
- Cache `XmlSerializer` instances.
- XML is inherently more verbose and slower to parse than JSON; prefer JSON where you control both ends.

---

## Summary

- **`XDocument`/LINQ to XML** — the modern, query-friendly XML API.
- **`XmlReader`/`XmlWriter`** — forward-only streaming for huge XML.
- **`XmlSerializer`** — opt-out POCO mapping (cache instances!); **`DataContractSerializer`** — opt-in, WCF-era.
- Mind XML namespaces (top cause of empty queries) and XXE security for untrusted input.
- **YAML** needs YamlDotNet (no built-in support).
- Prefer JSON for new APIs/config; XML for legacy/document interop; YAML for human-edited config.

→ Next: [07-Encoding.md](07-Encoding.md)
