+++
author = "yuhao"
title = "Generating QR Codes in C#"
date = "2025-01-01"
description = "There are multiple ways to generate QR codes. This article demonstrates how to use QRCoder—a lightweight and easy-to-use library that supports concurrent generation without relying on third-party..."
tags = [
    ".NET",
    ".NET Core",
    "QR Code",
]
categories = [
    "Programming/Development",
]
+++
There are multiple ways to generate QR codes. This article demonstrates how to use `QRCoder`—a lightweight and easy-to-use library that supports concurrent generation without relying on third-party dependencies.



---

## Setup

First, add the QRCoder package to your project via PowerShell:

```powershell
Install-Package QRCoder
```

---

## Implementation

### Define an Interface and Implementation

Create an `IQRCode` interface and its corresponding class to generate QR codes:

```csharp
using QRCoder;
using System.Drawing;
using System.Drawing.Imaging;
using System.IO;

public interface IQRCode
{
    byte[] GenerateQRCode(string content);
}

public class QRCode : IQRCode
{
    public byte[] GenerateQRCode(string content)
    {
        var generator = new QRCodeGenerator();
        var codeData = generator.CreateQrCode(content, QRCodeGenerator.ECCLevel.M, true);
        QRCoder.QRCode qrcode = new QRCoder.QRCode(codeData);

        // Generate bitmap image and convert to byte array
        var bitmapImg = qrcode.GetGraphic(
            pixelsPerModule: 10,
            darkColor: Color.Black,
            lightColor: Color.White,
            drawQuietZones: false
        );

        using MemoryStream stream = new MemoryStream();
        bitmapImg.Save(stream, ImageFormat.Jpeg);
        return stream.ToArray();
    }
}
```

#### Key Parameters of `GetGraphic()`

• `pixelsPerModule`: Pixel size per QR code module (larger values increase image resolution).
• `darkColor`: Color of the QR code's dark modules (typically `Color.Black`).
• `lightColor`: Color of the QR code's light modules (typically `Color.White`).
• `icon`: Optional watermark image (e.g., a logo) to embed in the QR code center.
• `iconSizePercent`: Size percentage of the watermark relative to the QR code.
• `iconBorderWidth`: Border width around the watermark.
• `drawQuietZones`: Whether to include empty margins around the QR code to improve readability.

---

### API Controller Integration

Inject the `IQRCode` service into an API controller to return the QR code as an image:

```csharp
using Microsoft.AspNetCore.Mvc;

[Route("api/[controller]")]
[ApiController]
public class QrCodeController : ControllerBase
{
    [HttpGet]
    public FileContentResult QrCode(
        [FromServices] IQRCode _qrcode, 
        string content
    )
    {
        var buffer = _qrcode.GenerateQRCode(content);
        return File(buffer, "image/jpeg");
    }
}
```

---

## Usage

• **Plain Text**: If `content` is plain text (e.g., `"Hello, World!"`), scanning the QR code will display the text.
• **URL**: If `content` is a valid URL (e.g., `"https://example.com"`), scanning will open the link directly.

**Example Output**
When the API is called with a URL, the generated QR code image will enable quick redirection to the specified webpage. Below is a textual representation of the QR code's expected output format (image omitted for brevity).

```
[Sample QR Code Image: Contains black modules on white background encoding the provided content]
```

---

## Advantages

• **Simplicity**: Minimal code required to generate QR codes.
• **Customization**: Adjust QR code colors, size, and embed logos.
• **Integration**: Easily integrates with ASP.NET Core for API-driven applications.

By leveraging `QRCoder`, you can efficiently generate QR codes tailored to your application's needs.

---

**GitHub Repository**: [https://github.com/codebude/QRCoder](https://github.com/codebude/QRCoder)

