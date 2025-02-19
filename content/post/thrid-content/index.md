+++
author = "yuhao"
title = "Implement Batch Dependency Injection in .NET 6 WebAPI"
date = "2025-02-17"
description = "dependency-injection"
tags = [
    "C#",
    ".NET",
    "string"
]
categories = [
    "Programming/Development",
    "Tutorials",
    "Practices"
]
series = ["tips and tricks"]
aliases = ["string-manipulation"]
image = "cover.jpg"
+++

## Preface

In .NET WebAPI development, when dealing with numerous business modules and classes that require dependency injection, manually registering each service becomes tedious and harms code readability. This article demonstrates an elegant batch injection solution using custom attributes.

## Implementation

### 1. Create Custom Attribute

Create an `AppServiceAttribute.cs` file:

```csharp
public class AppServiceAttribute : Attribute
{
    public ServiceLifeType ServiceLifeType { get; set; } = ServiceLifeType.Singleton;
    public Type ServiceType { get; set; }
}

public enum ServiceLifeType
{
    Transient,
    Scoped,
    Singleton
}
```

### 2. Batch Registration Extension

Create `AppServiceExtensions.cs`:

```csharp
public static class AppServiceExtensions
{
    public static void AddAppServices(this IServiceCollection services, params string[] assemblyNames)
    {
        foreach (var name in assemblyNames)
        {
            var assembly = Assembly.Load(name);
            foreach (var type in assembly.GetTypes())
            {
                var attribute = type.GetCustomAttribute<AppServiceAttribute>();
                if (attribute != null)
                {
                    RegisterService(services, type, attribute);
                }
            }
        }
    }

    private static void RegisterService(IServiceCollection services, Type implementationType, AppServiceAttribute attribute)
    {
        var serviceType = attribute.ServiceType ?? implementationType;

        switch (attribute.ServiceLifeType)
        {
            case ServiceLifeType.Singleton:
                services.AddSingleton(serviceType, implementationType);
                break;
            case ServiceLifeType.Scoped:
                services.AddScoped(serviceType, implementationType);
                break;
            case ServiceLifeType.Transient:
                services.AddTransient(serviceType, implementationType);
                break;
            default:
                services.AddTransient(serviceType, implementationType);
                break;
        }
    }
}
```

### 3. Service Implementation

`ISystemService.cs`:

```csharp
public interface ISystemService
{
    int Add(int a, int b);
}
```

`SystemService.cs`:

```csharp
[AppService(ServiceType = typeof(ISystemService), ServiceLifeType = ServiceLifeType.Scoped)]
public class SystemService : ISystemService
{
    public int Add(int a, int b) => a + b;
}
```

### 4. Registration in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAppServices("Your.Service.Assembly");
```

## Key Features

1. **Flexible Lifetime Management**: Easily configure service lifetime via attribute
2. **Interface-Implementation Decoupling**: Explicitly specify service contracts
3. **Batch Processing**: Automatically discover and register all decorated classes
4. **Code Generation Friendly**: Perfect match with T4 templates or Roslyn-based generators

## Recommended Improvements

1. Add null validation for `ServiceType`
2. Support automatic interface discovery (e.g., `IService` -> `Service`)
3. Add duplicate registration detection
4. Implement assembly scanning with `IAssemblyProvider`

> **Why Not Just Use Manual Registration?**  
> While this approach still requires attributes, it becomes extremely powerful when combined with code generators - most boilerplate code can be automatically generated during build process.

```

This implementation provides:
✅ Clean architecture separation
✅ Centralized dependency management
✅ Improved code maintainability
✅ Reduced human error in registration
```
