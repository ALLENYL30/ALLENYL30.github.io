+++
author = "yuhao"
title = "Web Scraping in .NET Core with HtmlAgilityPack, AngleSharp, and PuppeteerSharp"
date = "2024-09-27"
description = "Web scraping carries inherent risks and should be performed ethically. This guide demonstrates practical implementations using three popular .NET libraries."
tags = [
    ".NET",
    ".NET Core",
    "Web Scraping",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
Web scraping carries inherent risks and should be performed ethically. This guide demonstrates practical implementations using three popular .NET libraries.

---

## Core Components

### 1. Base Interface and Model

```csharp
public interface IHotNews
{
    Task<IList<HotNews>> GetHotNewsAsync();
}

public class HotNews
{
    public string Title { get; set; }
    public string Url { get; set; }
}
```

---

## Implementation Examples

### 1. HtmlAgilityPack

**Official Resources**:

- [Website](https://html-agility-pack.net/)
- [GitHub](https://github.com/zzzprojects/html-agility-pack)

**Installation**:

```powershell
Install-Package HtmlAgilityPack
```

**Blog Post Scraper**:

```csharp
public class HotNewsHtmlAgilityPack : IHotNews
{
    public async Task<IList<HotNews>> GetHotNewsAsync()
    {
        var web = new HtmlWeb();
        var doc = await web.LoadFromWebAsync("https://www.cnblogs.com/");
      
        return doc.DocumentNode.SelectNodes("//*[@id='post_list']/article/section/div/a")
            .Select(node => new HotNews 
            { 
                Title = node.InnerText, 
                Url = node.GetAttributeValue("href", "") 
            }).ToList();
    }
}
```

**Console Output Example**:

```
24 articles scraped
[Title 1]    https://example.com/post1
[Title 2]    https://example.com/post2
...
```

---

### 2. AngleSharp

**Official Resources**:

- [Website](https://anglesharp.github.io/)
- [GitHub](https://github.com/AngleSharp/AngleSharp)

**Installation**:

```powershell
Install-Package AngleSharp
```

**CSS Selector Implementation**:

```csharp
public class HotNewsAngleSharp : IHotNews
{
    public async Task<IList<HotNews>> GetHotNewsAsync()
    {
        var config = Configuration.Default.WithDefaultLoader();
        var context = BrowsingContext.New(config);
        var doc = await context.OpenAsync("https://www.cnblogs.com");
      
        return doc.QuerySelectorAll("article.post-item")
            .Select(item => new HotNews
            {
                Title = item.QuerySelector("section>div>a").TextContent,
                Url = item.QuerySelector("section>div>a").GetAttribute("href")
            }).ToList();
    }
}
```

**Output Verification**:
Same structured output as HtmlAgilityPack implementation.

---

### 3. PuppeteerSharp (SPA Support)

**Official Resources**:

- [Website](https://www.puppeteersharp.com/)
- [GitHub](https://github.com/hardkoded/puppeteer-sharp)

**Installation**:

```powershell
Install-Package PuppeteerSharp
```

**Core Workflow**:

```csharp
// Initialize browser
await new BrowserFetcher().DownloadAsync(BrowserFetcher.DefaultRevision);
using var browser = await Puppeteer.LaunchAsync(new LaunchOptions { Headless = true });

// Scrape SPA content
var page = await browser.NewPageAsync();
await page.GoToAsync("https://juejin.im", WaitUntilNavigation.Networkidle0);

// Example 1: Get full HTML
var html = await page.GetContentAsync();

// Example 2: Save screenshot
await page.ScreenshotAsync("juejin.png");

// Example 3: Generate PDF
await page.PdfAsync("juejin.pdf");
```

**Key Features**:

- Handles JavaScript-rendered pages
- Automated browser interactions
- File export capabilities (PNG/PDF)

---

## Execution Setup

```csharp
// Program.cs
static async Task Main(string[] args)
{
    var services = new ServiceCollection()
        .AddSingleton<IHotNews, HotNewsHtmlAgilityPack>() // Switch implementations
        .BuildServiceProvider();

    var scraper = services.GetRequiredService<IHotNews>();
    var results = await scraper.GetHotNewsAsync();

    Console.WriteLine($"Scraped {results.Count} items:");
    results.ForEach(item => Console.WriteLine($"{item.Title.PadRight(50)}\t{item.Url}"));
}
```

---

## Key Considerations

1. **Legality**: Always verify website scraping policies (check `robots.txt`)
2. **Rate Limiting**: Implement delays between requests
3. **Data Parsing**: Combine XPath/CSS selectors with regex for complex extraction
4. **Error Handling**: Use try-catch blocks for network instability
5. **SPA Handling**: PuppeteerSharp requires Chrome runtime (~150MB download)

For production scenarios, consider:

- Proxy rotation
- User-agent randomization
- Headless browser pooling

Complete code samples available on [GitHub](https://github.com/example/scraping-demo).

