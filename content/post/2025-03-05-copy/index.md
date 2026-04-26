+++
author = "yuhao"
title = "Three Methods to Copy Directories in C#"
date = "2025-03-04"
description = "Copying directories in C# isn’t straightforward since .NET lacks a built-in method for this task. This guide explores three practical approaches to achieve directory copying, each with its pros and..."
tags = [
    ".NET",
    ".NET Core",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
Copying directories in C# isn’t straightforward since .NET lacks a built-in method for this task. This guide explores three practical approaches to achieve directory copying, each with its pros and cons.

---

## Method 1: Recursive Copy

This method recursively copies files and subdirectories. It’s intuitive and aligns with Microsoft’s recommended pattern but requires careful handling of nested folders.

```csharp
static void CopyDirectory(string sourceFolderPath, string targetFolderPath)
{
    Directory.CreateDirectory(targetFolderPath);

    foreach (string filePath in Directory.GetFiles(sourceFolderPath))
    {
        string fileName = Path.GetFileName(filePath);
        string destinationPath = Path.Combine(targetFolderPath, fileName);
        File.Copy(filePath, destinationPath, true);
    }

    foreach (string directoryPath in Directory.GetDirectories(sourceFolderPath))
    {
        string directoryName = Path.GetFileName(directoryPath);
        string destinationPath = Path.Combine(targetFolderPath, directoryName);
        CopyDirectory(directoryPath, destinationPath);
    }
}
```

**Key Points**:
• `Directory.CreateDirectory` automatically creates missing parent directories.
• Recursion handles nested subdirectories.

---

## Method 2: Non-Recursive Copy

Avoid recursion by calculating relative paths for all files and reconstructing the directory hierarchy.

```csharp
static void CopyDirectory(string sourceFolderPath, string targetFolderPath)
{
    Directory.CreateDirectory(targetFolderPath);

    foreach (string filePath in Directory.GetFiles(sourceFolderPath, "*.*", SearchOption.AllDirectories))
    {
        string relativePath = Path.GetRelativePath(sourceFolderPath, filePath);
        string targetFilePath = Path.Combine(targetFolderPath, relativePath);
        string? subTargetFolderPath = Path.GetDirectoryName(targetFilePath);
    
        if (subTargetFolderPath != null)
            Directory.CreateDirectory(subTargetFolderPath);
    
        File.Copy(filePath, targetFilePath);
    }
}
```

**Key Points**:
• Uses `SearchOption.AllDirectories` to list all files recursively.
• `Path.GetRelativePath` computes the relative path for reconstructing the target hierarchy.
• Handle `Path.GetDirectoryName` returning `null` for root directories.

---

## Method 3: VisualBasic Built-in Method

Leverage the `Microsoft.VisualBasic.FileIO` namespace for simplicity, but note platform limitations.

```csharp
using Microsoft.VisualBasic.Devices;
using Microsoft.VisualBasic.FileIO;

static void CopyDirectory(string sourceFolderPath, string targetFolderPath)
{
    var fs = new Computer().FileSystem;
    fs.CopyDirectory(sourceFolderPath, targetFolderPath, UIOption.OnlyErrorDialogs);
}
```

**Key Points**:
• **Dependency**: Requires `Microsoft.VisualBasic` and Windows-specific frameworks (WPF/WinForms).
• **Limitation**: Not suitable for cross-platform projects (e.g., ASP.NET Core, Avalonia).

---

## Comparison and Recommendations

| Method | Pros | Cons | Use Case |
| - | - | - | - |
| **Recursive** | Simple, no external dependencies | Stack overflow risk for deep nesting | Small to medium directories |
| **Non-Recursive** | Avoids recursion pitfalls | Slightly complex path handling | Large or deeply nested structures |
| **VisualBasic** | Minimal code, built-in | Windows-only, requires WinForms | Windows desktop apps |

---

## Important Considerations

1. **Permissions**: Ensure read access to the source directory and write access to the target.
2. **File Conflicts**: Existing files in the target directory may be overwritten (controlled via `File.Copy`’s `overwrite` parameter).
3. **Performance**: For large directories, consider asynchronous operations or progress tracking.

---

## Conclusion

Choose the method based on your project’s needs:
• **Recursive**: Best for simplicity in small projects.
• **Non-Recursive**: Ideal for avoiding recursion limits.
• **VisualBasic**: Optimal for Windows-only applications with WinForms/WPF.

Always validate directory permissions and handle exceptions (e.g., `UnauthorizedAccessException`, `IOException`).

