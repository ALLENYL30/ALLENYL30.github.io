+++
author = "yuhao"
title = "Advanced XML Parsing: A Performance Duel Between LINQ, XPath, and Regex"
date = "2025-05-26"
description = "Continuing our exploration from last time, let's dive deeper into the surprising performance differences in XML parsing. Our sample XML remains the same, provided by W3Schools:"
tags = [
    "C#",
    ".NET",
    "XML",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
Continuing our exploration from last time, let's dive deeper into the surprising performance differences in XML parsing. Our sample XML remains the same, provided by W3Schools:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<breakfast_menu>
  <food>
    <name>Belgian Waffles</name>
    <price>$5.95</price>
    <description>Two of our famous Belgian Waffles with plenty of real maple syrup</description>
    <calories>650</calories>
  </food>
  <!-- Three other <food> elements omitted for brevity -->
  <food>
    <name>Homestyle Breakfast</name>
    <price>$6.95</price>
    <description>Two eggs, bacon or sausage, toast, and our ever-popular hash browns</description>
    <calories>950</calories>
  </food>
</breakfast_menu>
```

This time, our task is more specific: **get the value of the `<calories>` element from the *last* `<food>` node** (which should be `950`).

Our contestants are: LINQ to XML, XPath (running on `XDocument`), and Regular Expressions.

### LINQ to XML

First, let's see how to accomplish this using the `System.Xml.Linq` namespace.

```csharp
public int GetWithElements()
{
    var foods = doc.Root.Elements("food");
    var lastFood = foods.Last();
    return (int)lastFood.Element("calories");
}
```

Since we know the structure of the XML document, we can simplify this for a slight performance gain by not checking element names at every step:

```csharp
public int GetWithElementsOptimized()
    => (int)doc.Root.Elements().Last().Elements().Last();
```

We can also use the `Descendants()` method to reduce the number of `Elements()` calls. In this specific case, because the element we want happens to be the very last element in the entire document, we can take an extreme shortcut:

```csharp
public int GetWithDescendants()
    => (int)doc.Root.Descendants().Last();
```

### XPath

Next, let's try using an XPath expression. We can use the `System.Xml.XPath` namespace extension methods on our `XDocument` object.

```csharp
public int GetWithXPath()
    => (int)doc.XPathSelectElement("//food[last()]/calories");
```

Here, we use XPath's `last()` function to directly select the `<calories>` child of the last `<food>` element. We could have also hardcoded `[5]` instead of `[last()]` since we know there are five `<food>` elements. While this might provide a minuscule performance boost, it's not robust and feels a bit like cheating, so we'll stick with `last()`.

However, the expression above is not the most efficient. The `//food` syntax triggers a search for `<food>` elements across the *entire* XML document. We can significantly improve performance by writing a more precise path:

```csharp
public int GetWithXPathOptimized()
    => (int)doc.XPathSelectElement("/breakfast_menu/food[last()]/calories");
```

By specifying the full path, we instruct the engine to only look for `<food>` nodes that are direct children of the `<breakfast_menu>` root. This seemingly small change yields a significant performance boost (around **4-5 times faster!**).

### Regular Expressions

Finally, let's look at the simple and direct approach with Regex. We just need to match all `<calories>` nodes and grab the value from the last match.

```csharp
private readonly Regex regex = new Regex(@"<calories>(\d+)</calories>");

public int GetWithRegex()
{
    var matches = regex.Matches(xml);
    return int.Parse(matches[^1].Groups[1].Value);
}
```

But there's a huge opportunity for optimization here. Since we only need the *last* match, we don't need to find all of them. We can achieve this by using the `RegexOptions.RightToLeft` option, which tells the engine to start its search from the end of the string.

```csharp
private readonly Regex regexOptimized = new Regex(@"<calories>(\d+)</calories>", RegexOptions.RightToLeft);

public int GetWithRegexOptimized()
{
    var match = regexOptimized.Match(xml);
    return int.Parse(match.Groups[1].Value);
}
```

With this simple change, we again achieve a performance improvement of about **4-5 times**.

### Performance Comparison

Now, let's look at the results of our race. Here's a summary of the benchmark findings, from fastest to slowest:

1. **`Elements()` (LINQ to XML):** ~101 ns
2. **`RegexOptimized()` (RightToLeft):** ~128 ns
3. **`Descendants()` (LINQ to XML):** ~243 ns
4. **`Regex()` (Standard):** ~605 ns
5. **`XPathOptimized()` (Full Path):** ~1,297 ns
6. **`XPath()` (Global Search):** ~5,099 ns

Are these results what you expected? It's surprising to see that the humble **LINQ to XML methods easily outperformed even the optimized Regex and XPath versions.** Most notably, XPath was shockingly slow, with its execution time measured in microseconds, not nanoseconds.

Meanwhile, regular expressions—the winner in our previous contest—couldn't keep up with LINQ to XML this time. And without the `RightToLeft` optimization, the Regex performance was significantly worse.

This benchmark proves once again that for querying XML documents, **LINQ to XML is often the best choice**, offering a fantastic combination of ease-of-use and high performance.

### One More Thing...

No performance showdown would be complete without an entry from `Span<T>`.

```csharp
public int GetWithSpan()
{
	var xmlSpan = testXml.AsSpan();
	int endTagIndex = xmlSpan.LastIndexOf("</calories>");
	var trimmedSpan = xmlSpan.Slice(0, endTagIndex);
	int startTagIndex = trimmedSpan.LastIndexOf("<calories>") + 10;
	return int.Parse(xmlSpan.Slice(startTagIndex, endTagIndex - startTagIndex));
}
```

And the result?

* **`GetWithSpan()` Mean Execution Time:** **~22.3 ns**

As expected, `Span<T>` operates on another level entirely, leaving all other methods in the dust and showcasing its power for raw, low-level text processing.
