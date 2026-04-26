+++
author = "yuhao"
title = "Configuration Management in .NET Core with appsettings.json"
date = "2025-02-03"
description = "A comprehensive guide to leveraging appsettings.json for flexible configuration management in .NET Core applications."
tags = [
    ".NET",
    ".NET Core",
    "C#",
]
categories = [
    "Programming/Development",
]
+++
A comprehensive guide to leveraging appsettings.json for flexible configuration management in .NET Core applications.

## Quick Start

### 1. Create appsettings.json

```json
{
  "AppSettings": {
    "LogLevel": "Warning",
    "ConnectionStrings": {
      "Default": "Server=localhost;Database=mydb;"
    }
  }
}
```

### 2. Basic Configuration Loading

```csharp
using Microsoft.Extensions.Configuration;

var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();

string logLevel = config["AppSettings:LogLevel"];
string connectionString = config["AppSettings:ConnectionStrings:Default"];
```

## Core Features

### 1. Value Retrieval Methods

```csharp
// Traditional string access
var logLevel = config["AppSettings:LogLevel"];

// Type-safe retrieval
int timeout = config.GetValue<int>("Network:Timeout", 30); // Default value

// Connection string shortcut
var dbConn = config.GetConnectionString("Default");
```

### 2. Multi-Environment Configuration

**appsettings.Development.json**

```json
{
  "AppSettings": {
    "LogLevel": "Debug",
    "DebugMode": true
  }
}
```

**Configuration Builder**

```csharp
var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json") // Base configuration
    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
    .Build();
```

*Configuration Priority*:

1. Command-line arguments
2. Environment variables
3. Environment-specific JSON
4. Base appsettings.json

## Advanced Techniques

### 1. Strongly-Typed Configuration

**Configuration Class**

```csharp
public class AppSettings
{
    public string LogLevel { get; set; }
    public DatabaseConfig Database { get; set; }

    public class DatabaseConfig
    {
        public string Host { get; set; }
        public int Port { get; set; }
    }
}
```

**Binding Configuration**

```csharp
var appConfig = config.GetSection("AppSettings").Get<AppSettings>();
Console.WriteLine($"DB Port: {appConfig.Database.Port}");
```

### 2. Dynamic Reloading

```csharp
var reloadToken = config.GetReloadToken();
reloadToken.RegisterChangeCallback(_ => 
{
    Console.WriteLine("Configuration changed!");
    // Refresh configuration-sensitive components
}, null);
```

## Runtime Configuration Sources

### 1. Environment Variables

```bash
# Linux/macOS
export AppSettings__LogLevel=Verbose

# Windows
set AppSettings__LogLevel=Verbose
```

### 2. Command-Line Arguments

```csharp
var config = new ConfigurationBuilder()
    .AddCommandLine(args)
    .Build();
```

**Execution Example**

```bash
dotnet run --AppSettings:LogLevel=Critical
```

## Best Practices

1. **Security Considerations**

```csharp
// Never store secrets in appsettings.json
var secret = config["Azure:KeyVaultSecret"];
```

2. **Validation Pattern**

```csharp
services.Configure<AppSettings>(config.GetSection("AppSettings"))
        .ValidateDataAnnotations();
```

3. **Custom Configuration Providers**

```csharp
public class DatabaseConfigurationProvider : ConfigurationProvider
{
    public override void Load()
    {
        // Load config from database
        Data = GetConfigFromDB();
    }
}
```

## Key Limitations

- **No Native Write Support**: Configuration system is read-only
- **Type Conversion Requirements**: Manual handling for complex types
- **Environment-Specific Complexity**: Requires careful management of multiple files

---

**Recommended NuGet Packages**

- `Microsoft.Extensions.Configuration.Json`
- `Microsoft.Extensions.Configuration.EnvironmentVariables`
- `Microsoft.Extensions.Configuration.CommandLine`
- `Microsoft.Extensions.Options.ConfigurationExtensions`

[Official Configuration Documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/)
[Configuration Best Practices](https://docs.microsoft.com/en-us/dotnet/core/extensions/configuration)

> **Next Steps**: Explore integration with HostBuilder and dependency injection for enterprise-level configuration management.

