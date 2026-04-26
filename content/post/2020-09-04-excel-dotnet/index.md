+++
author = "yuhao"
title = "Excel Import and Export Using EPPlus in .Net"
date = "2020-09-03"
description = "EPPlus GitHub Repository: https://github.com/EPPlusSoftware/EPPlus"
tags = [
    ".NET",
    ".NET Core",
    "EPPlus",
    "Excel",
]
categories = [
    "Programming/Development",
]
+++
**EPPlus GitHub Repository**: [https://github.com/EPPlusSoftware/EPPlus](https://github.com/EPPlusSoftware/EPPlus)

Add the EPPlus package to your project:

```powershell
Install-Package EPPlus
```

---

## 📥 Excel Import Example

Read data from an Excel file and serialize it into JSON.

**Sample Excel Structure**:

```
| AAA | BBB | CCC | DDD | EEE | FFF |
|-----|-----|-----|-----|-----|-----|
| A1  | B1  | C1  | D1  | E1  | F1  |
| A2  | B2  | C2  | D2  | E2  | F2  |
```

```csharp
[HttpPost]
public List<ExcelDemoDto> Import([FromForm] ImportExcelInput input)
{
    var list = new List<ExcelDemoDto>();

    using (var package = new ExcelPackage(input.ExcelFile.OpenReadStream()))
    {
        var sheet = package.Workbook.Worksheets.First();
        
        int startRow = sheet.Dimension.Start.Row + 1;
        int endRow = sheet.Dimension.End.Row;
        
        for (int row = startRow; row <= endRow; row++)
        {
            list.Add(new ExcelDemoDto
            {
                AAA = sheet.Cells[row, 1].Text,
                BBB = sheet.Cells[row, 2].Text,
                CCC = sheet.Cells[row, 3].Text,
                DDD = sheet.Cells[row, 4].Text,
                EEE = sheet.Cells[row, 5].Text,
                FFF = sheet.Cells[row, 6].Text
            });
        }
    }
    return list;
}

// DTO Class
public class ExcelDemoDto
{
    public string AAA { get; set; }
    public string BBB { get; set; }
    public string CCC { get; set; }
    public string DDD { get; set; }
    public string EEE { get; set; }
    public string FFF { get; set; }
}

// Input Model
public class ImportExcelInput
{
    public IFormFile ExcelFile { get; set; }
}
```

**JSON Output Example**:

```json
[
  {"AAA":"A1","BBB":"B1","CCC":"C1","DDD":"D1","EEE":"E1","FFF":"F1"},
  {"AAA":"A2","BBB":"B2","CCC":"C2","DDD":"D2","EEE":"E2","FFF":"F2"}
]
```

---

## 📤 Excel Export Example

Generate an Excel file with mock data and save it locally.

```csharp
[HttpGet]
public async Task<string> Export()
{
    using var package = new ExcelPackage();
    var worksheet = package.Workbook.Worksheets.Add("Sheet1");

    // Set headers
    string[] headers = { "AAA", "BBB", "CCC", "DDD", "EEE", "FFF" };
    for (int i = 0; i < headers.Length; i++)
    {
        worksheet.Cells[1, i + 1].Value = headers[i];
        worksheet.Cells[1, i + 1].Style.Font.Bold = true;
    }

    // Mock data
    var data = Enumerable.Range(1, 10).Select(i => new ExcelDemoDto
    {
        AAA = $"A{i}", BBB = $"B{i}", CCC = $"C{i}",
        DDD = $"D{i}", EEE = $"E{i}", FFF = $"F{i}"
    }).ToList();

    // Populate data
    int row = 2;
    foreach (var item in data)
    {
        worksheet.Cells[row, 1].Value = item.AAA;
        worksheet.Cells[row, 2].Value = item.BBB;
        worksheet.Cells[row, 3].Value = item.CCC;
        worksheet.Cells[row, 4].Value = item.DDD;
        worksheet.Cells[row, 5].Value = item.EEE;
        worksheet.Cells[row, 6].Value = item.FFF;
        row++;
    }

    // Save file
    string path = Path.Combine(Directory.GetCurrentDirectory(), "export.xlsx");
    await package.GetAsByteArray().DownloadAsync(path);
    return $"File saved to: {path}";
}
```

**Exported Excel Preview**:

```
Bold headers: AAA | BBB | CCC | DDD | EEE | FFF
Row 2: A1 | B1 | C1 | D1 | E1 | F1
...
Row 11: A10 | B10 | C10 | D10 | E10 | F10
```

---

Key features demonstrated:

- Stream-based Excel file handling
- Dynamic row/column detection
- Header styling (bold text)
- File export to local storage
- Compatibility with ASP.NET Core file uploads

