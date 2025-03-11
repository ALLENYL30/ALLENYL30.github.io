+++
author = "yuhao"
title = "Common Image Data Type Conversions in .NET Development"
date = "2025-02-20"
description = "Data Type Conversions"
tags = [
    "C#",
    ".NET",
    "image"
]
categories = [
    "Programming/Development"
]
series = ["tips and tricks"]
aliases = ["image data type Conversions"]
image = "cover.jpg"
+++

When developing with .NET, especially WPF applications, handling various image data types is a frequent requirement. The "type" mentioned here refers to how image data is represented in memory rather than the image file formats (e.g., JPG, PNG, BMP). Converting between these types can sometimes be confusing. This document summarizes common conversion methods to clarify these processes.

## Common Image Data Types

- **byte[]**: Byte arrays, either:
  - Raw data read from image files (including headers).
  - Pixel data such as RGB values.
- **Stream**: Data streams like `MemoryStream` or `FileStream`, easily convertible to and from byte arrays.
- **Bitmap**: Used in WinForms (based on GDI+), located in `System.Drawing` namespace.
- **BitmapImage**: Used in WPF (`System.Windows.Media.Imaging`), commonly set as the `Source` for `Image` controls.
- **BitmapSource**: Base class for `BitmapImage`, used in WPF (`System.Windows.Media`).
- Other image types from third-party libraries.

## Conversion Examples

### Convert Image Path to BitmapImage

If you have an image path (local or URL) and wish to display it in an `Image` control:

```csharp
var image = new Image();
image.Source = new BitmapImage(new Uri(@"path\to\image.jpg", UriKind.RelativeOrAbsolute));
```

This method automatically handles common formats (JPG, PNG, BMP, GIF, TIFF, WebP, HEIC, AVIF). Uncommon formats may require third-party libraries.

### Convert Bitmap to BitmapImage

Converting from WinForms (`Bitmap`) to WPF (`BitmapImage`):

```csharp
using System.Drawing;
using System.Windows.Media.Imaging;

static BitmapImage ConvertBitmapToBitmapImage(Bitmap bitmap)
{
    using var stream = new MemoryStream();
    bitmap.Save(stream, ImageFormat.Png);
    stream.Position = 0;

    var bitmapImage = new BitmapImage();
    bitmapImage.BeginInit();
    bitmapImage.CacheOption = BitmapCacheOption.OnLoad;
    bitmapImage.StreamSource = stream;
    bitmapImage.EndInit();
    bitmapImage.Freeze(); // Optional: enhances performance and thread safety
    return bitmapImage;
}
```

### Convert Byte Array to ImageSource

#### From file data (e.g., JPG, PNG, BMP):

```csharp
using System.IO;
using System.Windows.Media.Imaging;

static ImageSource ConvertByteArrayToImageSource(byte[] bytes)
{
    using var stream = new MemoryStream(bytes);
    var bitmapImage = new BitmapImage();
    bitmapImage.BeginInit();
    bitmapImage.CacheOption = BitmapCacheOption.OnLoad;
    bitmapImage.StreamSource = stream;
    bitmapImage.EndInit();
    bitmapImage.Freeze();
    return bitmapImage;
}
```

Alternatively, using `BitmapDecoder`:

```csharp
using System.Windows.Media.Imaging;

static ImageSource ConvertByteArrayToImageSource(byte[] bytes)
{
    using var stream = new MemoryStream(bytes);
    return BitmapDecoder
        .Create(stream, BitmapCreateOptions.PreservePixelFormat, BitmapCacheOption.OnLoad)
        .Frames[0];
}
```

#### From pixel data (e.g., RGB data):

```csharp
using System.Windows.Media;
using System.Windows.Media.Imaging;

static ImageSource BgrByteArrayToImageSource(byte[] array, int width, int height, int channel = 3, int? stride = null)
{
    var bmp = new WriteableBitmap(width, height, 96, 96, PixelFormats.Bgr24, null);
    stride ??= ((width * channel + 3) / 4) * 4;
    bmp.WritePixels(new Int32Rect(0, 0, width, height), array, stride.Value, 0);
    bmp.Freeze();
    return bmp;
}
```

### Convert BitmapSource to BitmapImage

Typically, converting from clipboard data (`BitmapSource`) to `BitmapImage`:

```csharp
using System.Windows.Media.Imaging;

static BitmapImage ConvertBitmapSourceToBitmapImage(BitmapSource bitmapSource)
{
    var bitmapImage = new BitmapImage();
    using var stream = new MemoryStream();
    BitmapEncoder encoder = new BmpBitmapEncoder(); // Clipboard images are typically BMP
    encoder.Frames.Add(BitmapFrame.Create(bitmapSource));
    encoder.Save(stream);
    stream.Position = 0;
    bitmapImage.BeginInit();
    bitmapImage.CacheOption = BitmapCacheOption.OnLoad;
    bitmapImage.StreamSource = stream;
    bitmapImage.EndInit();
    bitmapImage.Freeze();
    return bitmapImage;
}
```

### Convert Emgu.CV.Image to BitmapImage

For uncommon formats (e.g., JP2), use the Emgu.CV library:

```csharp
var filename = @"path\to\image.jp2";
var mat = new Image<Bgr, Byte>(filename);
var bytes = mat.ToJpegData();

using var stream = new MemoryStream(bytes);
var bitmap = new BitmapImage();
bitmap.BeginInit();
bitmap.CacheOption = BitmapCacheOption.OnLoad;
bitmap.StreamSource = stream;
bitmap.EndInit();
bitmap.Freeze();

var control = new Image();
control.Source = bitmap;
```

## Summary

Most conversions involve converting data to a common `Stream` format first, creating a `BitmapImage`, and then assigning it to an `Image` control. Following this pattern ensures consistent and effective image handling across various scenarios.
