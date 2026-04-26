+++
author = "yuhao"
title = "Exceptionless Logging in .NET Core"
date = "2024-09-12"
description = "ExceptionLess is a free open-source distributed logging framework that provides self-hosted and managed solutions for error tracking and log collection. Key features include:"
tags = [
    ".NET",
    ".NET Core",
    "Exceptionless",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
# Exceptionless Logging Framework Guide

## Overview

[ExceptionLess](https://exceptionless.com) is a **free open-source distributed logging framework** that provides self-hosted and managed solutions for error tracking and log collection. Key features include:

- Real-time error monitoring
- Structured logging capabilities
- Cross-platform client support
- Custom event categorization
- Powerful search/filter capabilities

**GitHub Repositories**:

- Main Project: [exceptionless/Exceptionless](https://github.com/exceptionless/Exceptionless)
- .NET Client: [exceptionless/Exceptionless.Net](https://github.com/exceptionless/Exceptionless.Net)

## Deployment Options

### Self-Hosted Installation (Docker)

```powershell
docker run --rm -it -p 5000:80 exceptionless/exceptionless:6.1.0
```

After successful startup:

1. Access dashboard: [http://localhost:5000](http://localhost:5000)
2. Create account and login
3. Create new project


> For production environments, see [Self-Hosting Guide](https://github.com/exceptionless/Exceptionless/wiki/Self-Hosting)

## .NET Core Integration

### Step 1: Install NuGet Package

```powershell
Install-Package Exceptionless.AspNetCore
```

### Step 2: Configuration

```json
// appsettings.json
{
  "Exceptionless": {
    "ServerUrl": "http://localhost:5000",
    "ApiKey": "YOUR_API_KEY_HERE"
  }
}
```

### Step 3: Initialize Middleware

```csharp
// Startup.cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Other middleware
    app.UseExceptionless(Configuration);
    // ...
}
```

## Usage Examples

### Automatic Exception Logging

All unhandled exceptions are automatically captured:

```csharp
// Example controller action
[HttpGet]
public IActionResult ErrorTest()
{
    throw new InvalidOperationException("Sample exception");
}
```

### Manual Error Reporting

```csharp
try
{
    // Application code
}
catch (Exception ex)
{
    ex.ToExceptionless().Submit();
}
```

## Dashboard Features


Key dashboard capabilities:

- Error frequency analysis
- Stack trace visualization
- Environment filtering (DEV/UAT/PROD)
- Custom event tagging
- Team collaboration features

## Advanced Configuration

| Setting | Description | Default |
|---------|-------------|---------|
| `ServerUrl` | Self-hosted instance URL | Managed service |
| `ApiKey` | Project authentication key | Required |
| `Enabled` | Enable/disable logging | true |
| `Tags` | Custom event tags | None |
| `OmitStackTrace` | Disable stack traces | false |

```csharp
// Advanced initialization
services.AddExceptionless(config => {
    config.ApiKey = Configuration["Exceptionless:ApiKey"];
    config.ServerUrl = Configuration["Exceptionless:ServerUrl"];
    config.SetVersion("1.2.0");
});
```

## Best Practices

1. **Environment Segregation**: Use different projects for DEV/UAT/PROD
2. **Error Classification**: Apply tags like "Critical", "UI-Error", etc.
3. **User Context**: Attach user information to events
   ```csharp
   ExceptionlessClient.Default.CreateLog("LoginFailed")
       .SetUserIdentity(user.Email)
       .Submit();
   ```
4. **Rate Limiting**: Configure event submission thresholds
5. **Data Retention**: Set appropriate retention policies

For full documentation: [Exceptionless.Net Wiki](https://github.com/exceptionless/Exceptionless.Net/wiki)

