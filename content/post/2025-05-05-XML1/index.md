+++
author = "yuhao"
title = "Efficient XML Parsing in C#: A Performance Comparison"
date = "2025-05-04"
description = "I recently encountered a task at work involving reading and writing XML, which led me to explore the performance implications of different approaches. I'd like to share my findings with you."
tags = [
    ".NET",
    "C#",
    ".Net",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
I recently encountered a task at work involving reading and writing XML, which led me to explore the performance implications of different approaches. I'd like to share my findings with you.

For our demonstration, we'll use a sample XML text from W3Schools:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<breakfast_menu>
  <food>
    <name>Belgian Waffles</name>
    <price>$5.95</price>
    <description>Two of our famous Belgian Waffles with plenty of real maple syrup</description>
    <calories>650</calories>
  </food>
  <!-- ... three <food> elements omitted for brevity ... -->
  <food>
    <name>Homestyle Breakfast</name>
    <price>$6.95</price>
    <description>Two eggs, bacon or sausage, toast, and our ever-popular hash browns</description>
    <calories>950</calories>
  </food>
</breakfast_menu>
```

Our goal is to read the values of all `<name>` elements and store them in a `List<string>`. Let's implement this using several different methods.

### Using `XmlDocument`

`XmlDocument` represents the traditional, DOM-based approach. It can be used in a couple of ways: one is by using methods like `GetElementsByTagName` to traverse the document tree, and the other is by using XPath expressions to select nodes directly. Let's look at the first method:

```csharp
public List<string> XmlDocument()
{
    var doc = new XmlDocument();
    doc.LoadXml(testXml);
    return doc
        .GetElementsByTagName("food")
        .OfType<XmlNode>()
        .Select(node => node["name"]!.InnerText)
        .ToList();
}
```

Now, let's try it with an XPath expression:

```csharp
public List<string> XmlDocumentXPath()
{
    var doc = new XmlDocument();
    doc.LoadXml(testXml);
    return doc
        .SelectNodes("//food/name")
        .OfType<XmlNode>()
        .Select(node => node.InnerText)
        .ToList();
}
```

### Using `System.Xml.Linq`

`System.Xml.Linq` (`XDocument`) offers a more modern and developer-friendly API. Here's how to implement our task with it:

```csharp
public List<string> XDocument()
{
    var doc = XDocument.Parse(testXml);
    return doc
        .Root
        .Elements("food")
        .Select(node => node.Element("name")!.Value)
        .ToList();
}
```

While `XDocument` also supports XPath, its fluent API is already so concise that using XPath often isn't necessary for simpler queries.

### Using `XmlReader`

`XmlReader` is a forward-only, stream-based parser. While its API is more complex to use, it is extremely efficient in terms of speed and memory.

```csharp
public List<string> XmlReader()
{
    using var stringReader = new StringReader(testXml);
    using var xmlReader = System.Xml.XmlReader.Create(stringReader);
    var res = new List<string>(8);
    while (xmlReader.Read())
    {
        if (xmlReader.IsStartElement() && xmlReader.Name == "name")
        {
            res.Add(xmlReader.ReadElementContentAsString());
        }
    }
    return res;
}
```

### Using Regular Expressions (Regex)

Since our task is relatively simple and the XML structure is predictable, we can also use regular expressions.

```csharp
public List<string> Regex()
{
    var matches = System.Text.RegularExpressions.Regex.Matches(testXml, @"<name>(.*?)</name>");
    return matches.Select(match => match.Groups[1].Value).ToList();
}
```

### Using Traditional String Methods

Finally, we can fall back on basic string manipulation methods.

```csharp
public List<string> StringOps()
{
    var res = new List<string>(8);
    int cur = 0;
    while (true)
    {
        // Find the next <name> tag
    	int idx = testXml.IndexOf("<name>", cur);
        // If not found, we're done
    	if (idx < 0)
    		break;
        // Find the corresponding </name> tag
    	int end = testXml.IndexOf("</name>", idx + 6);
    	res.Add(testXml.Substring(idx + 6, end - idx - 6));
        // Continue searching from the end of the current tag
    	cur = end + 7;
    }
    return res;
}
```

Spoiler alert: this method is surprisingly inefficient, much slower than the other approaches. This leads us to our secret weapon: `Span<T>`.

### Using `Span<T>`

`Span<T>`, introduced in C# 7.2, allows for highly efficient, allocation-free memory manipulation.

```csharp
public List<string> SpanOps()
{
    var res = new List<string>(8);
    var span = testXml.AsSpan();
    while (true)
    {
        int idx = span.IndexOf("<name>");
        if (idx < 0)
            break;
        // Slice from after "<name>" and find the closing tag
        var contentSlice = span.Slice(idx + 6);
        int end = contentSlice.IndexOf("</name>");
        // Add the content to the list
        res.Add(contentSlice.Slice(0, end).ToString());
        // Move the span past the content we just processed
        span = contentSlice.Slice(end + 7);
    }
    return res;
}
```

### Performance Test

We used **BenchmarkDotNet** to test the performance of these methods. Here are the results:

| Case        | Mean       | Min        | Max        | Allocated Bytes |
|-------------|------------|------------|------------|-----------------|
| `XmlDocument` | 10.65 µs   | 10.40 µs   | 10.79 µs   | 17,608          |
| `XPath`     | 5.74 µs    | 5.72 µs    | 5.76 µs    | 18,912          |
| `LINQ`      | 3.95 µs    | 3.94 µs    | 3.97 µs    | 15,054          |
| `XmlReader`   | 3.04 µs    | 3.02 µs    | 3.06 µs    | 11,776          |
| `Regex`     | 2.13 µs    | 2.11 µs    | 2.14 µs    | 2,632           |
| `StringOps`   | 37.16 µs   | 37.08 µs   | 37.32 µs   | 448             |
| `SpanOps`     | 134.48 ns  | 133.89 ns  | 135.11 ns  | 448             |

So, were you impressed by the power of `Span<T>`? It's in a league of its own, reaching nanosecond-level performance. From these results, we can draw a few conclusions:

* If the content we need to extract isn't complex and the structure is consistent, we can use **Regex** for a very fast and low-allocation solution.
* When the task requires robust XML parsing, we should rely on dedicated XML APIs. Their performance ranking is: **`XmlReader` > `XDocument` (LINQ) > `XmlDocument`**.
* From a practical standpoint, **`XDocument`** is often the best choice. It's significantly faster than the traditional `XmlDocument` and not much slower than `XmlReader`, but with a much more pleasant and productive API, making it the optimal choice for most scenarios.
* Using **`Span<T>`** can lead to massive performance gains, especially when your logic involves frequent string operations like `IndexOf` and slicing (`Substring`). It's the clear winner for performance-critical scenarios that can be solved with string manipulation.

