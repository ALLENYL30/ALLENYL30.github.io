+++
author = "yuhao"
title = "Implementing a Custom Captcha Generator in ASP.NET Core Without Third-Party Libraries"
date = "2024-12-03"
description = "A practical guide to creating random captcha images in .NET applications without external dependencies. This solution leverages System.Drawing.Common for image manipulation and provides full control..."
tags = [
    ".NET",
    ".NET Core",
    "验证码",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
A practical guide to creating random captcha images in .NET applications without external dependencies. This solution leverages `System.Drawing.Common` for image manipulation and provides full control over captcha generation.

## Core Implementation

### 1. Captcha Interface

```csharp
public interface ICaptcha
{
    /// <summary>
    /// Generates a random captcha code
    /// </summary>
    /// <param name="codeLength">Number of characters (default: 4)</param>
    Task<string> GenerateRandomCaptchaAsync(int codeLength = 4);

    /// <summary>
    /// Generates captcha image from specified code
    /// </summary>
    /// <param name="captchaCode">Pre-generated captcha text</param>
    /// <param name="width">Auto-calculated if 0</param>
    /// <param name="height">Image height</param>
    Task<CaptchaResult> GenerateCaptchaImageAsync(string captchaCode, int width = 0, int height = 30);
}
```

### 2. Data Model

```csharp
public class CaptchaResult
{
    /// <summary>
    /// Generated captcha text
    /// </summary>
    public string CaptchaCode { get; set; }

    /// <summary>
    /// Image stream in PNG format
    /// </summary>
    public MemoryStream CaptchaMemoryStream { get; set; }

    /// <summary>
    /// Generation timestamp
    /// </summary>
    public DateTime Timestamp { get; set; }
}
```

### 3. Captcha Generator Implementation

```csharp
using System;
using System.Drawing;
using System.Drawing.Imaging;
using System.IO;
using System.Threading.Tasks;

public class Captcha : ICaptcha
{
    private const string CharacterPool = "1,2,3,4,5,6,7,8,9,A,B,C,D,E,F,G,H,J,K,L,M,N,P,Q,R,S,T,U,V,W,X,Y,Z";

    public Task<CaptchaResult> GenerateCaptchaImageAsync(string captchaCode, int width = 0, int height = 30)
    {
        // Color palette for captcha text
        Color[] textColors = { Color.Black, Color.Red, Color.DarkBlue, 
                             Color.Green, Color.Orange, Color.Brown, 
                             Color.DarkCyan, Color.Purple };

        // Supported fonts for captcha generation
        string[] fonts = { "Verdana", "Microsoft Sans Serif", 
                         "Comic Sans MS", "Arial" };

        var image = new Bitmap(width == 0 ? captchaCode.Length * 25 : width, height);
        using var graphics = Graphics.FromImage(image);
      
        graphics.Clear(Color.White);
        var random = new Random();

        // Add background noise
        for (var i = 0; i < 100; i++)
        {
            graphics.DrawRectangle(Pens.LightGray, 
                random.Next(image.Width), 
                random.Next(image.Height), 
                1, 1);
        }

        // Draw each character
        for (var i = 0; i < captchaCode.Length; i++)
        {
            var font = new Font(fonts[random.Next(fonts.Length)], 15, FontStyle.Bold);
            var brush = new SolidBrush(textColors[random.Next(textColors.Length)]);

            var verticalOffset = (i + 1) % 2 == 0 ? 2 : 4;
            graphics.DrawString(captchaCode[i].ToString(), font, brush, 
                17 + (i * 17), verticalOffset);
        }

        var memoryStream = new MemoryStream();
        image.Save(memoryStream, ImageFormat.Png);

        return Task.FromResult(new CaptchaResult
        {
            CaptchaCode = captchaCode,
            CaptchaMemoryStream = memoryStream,
            Timestamp = DateTime.Now
        });
    }

    public Task<string> GenerateRandomCaptchaAsync(int codeLength = 4)
    {
        var characters = CharacterPool.Split(',');
        var random = new Random();
        var captcha = new char[codeLength];

        for (int i = 0; i < codeLength; i++)
        {
            captcha[i] = characters[random.Next(characters.Length)][0];
        }

        return Task.FromResult(new string(captcha));
    }
}
```

### 4. API Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class CaptchaController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GenerateCaptcha([FromServices] ICaptcha captchaService)
    {
        var code = await captchaService.GenerateRandomCaptchaAsync();
        var imageData = await captchaService.GenerateCaptchaImageAsync(code);
      
        return File(imageData.CaptchaMemoryStream.ToArray(), "image/png");
    }
}
```

## Usage Recommendations

1. **Validation Workflow**:
   
   - Store generated codes in session or distributed cache
   - Add expiration time (typically 3-5 minutes)
   - Compare user input with stored value case-insensitively
2. **Security Enhancements**:
   
   ```csharp
   // In Startup.cs
   services.AddSession(options => 
   {
       options.Cookie.HttpOnly = true;
       options.Cookie.SameSite = SameSiteMode.Strict;
   });
   
   services.AddDistributedMemoryCache();
   ```
3. **Performance Tips**:
   
   - Cache generated images
   - Implement rate limiting
   - Consider async I/O operations

**Example Output**:
A 150x30 pixel PNG image containing 4 distorted alphanumeric characters with multicolor text and background noise patterns.

---

**Key Features**:
✅ Pure .NET implementation
✅ Customizable appearance parameters
✅ Thread-safe generation
✅ Session-based validation support
✅ Lightweight (No external dependencies)

This implementation provides a balance between security and usability while maintaining full control over the captcha generation process.

