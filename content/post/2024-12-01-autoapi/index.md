+++
author = "yuhao"
title = "Auto-Generating RESTful APIs in .NET with Plus.AutoApi"
date = "2024-11-30"
description = "In DDD (Domain-Driven Design) architectures, developers often need to manually create API controllers after implementing service layers. Plus.AutoApi eliminates this step by dynamically generating..."
tags = [
    ".NET",
    ".NET Core",
    "WebApi",
]
categories = [
    "Programming/Development",
]
+++
## Introduction

In DDD (Domain-Driven Design) architectures, developers often need to manually create API controllers after implementing service layers. **Plus.AutoApi** eliminates this step by dynamically generating RESTful-style WebAPIs without requiring Controller classes.

## Quick Start

### 1. Install NuGet Package

Add the package to your application service layer:

```powershell
Install-Package Plus.AutoApi
```

### 2. Service Registration

Configure in `Startup.cs`:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddAutoApi(options => { });
}
```

### 3. Implement Service

Create service classes implementing `IAutoApi` with `[AutoApi]` attribute:

```csharp
[AutoApi]
public class WeatherService : IAutoApi
{
    private static readonly string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    // Auto-generated GET endpoint: /api/weather
    public IEnumerable<WeatherForecast> Get()
    {
        return GenerateForecast();
    }

    // Custom GET endpoint: /api/weather/{id}
    [HttpGet("{id}")]
    public IEnumerable<WeatherForecast> Get(int id)
    {
        return GenerateForecast();
    }

    private static IEnumerable<WeatherForecast> GenerateForecast()
    {
        var rng = new Random();
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = rng.Next(-20, 55),
            Summary = Summaries[rng.Next(Summaries.Length)]
        });
    }
}
```

### Key Features:

- **Naming Convention**: Services ending with `Service`/`ApplicationService` auto-generate APIs
- **Custom Endpoints**: Use standard ASP.NET Core attributes like `[HttpGet("{id}")]`
- **Opt-Out**: Disable generation with `[AutoApi(Disabled = true)]`

## Swagger Integration

Enable OpenAPI documentation:

```csharp
services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "AutoAPI Demo",
        Version = "v1.0.0"
    });
    options.DocInclusionPredicate((docName, description) => true); // Include all endpoints
});
```

## HTTP Method Convention

Default verb mapping based on method names:

| Method Prefix  | HTTP Verb |
|----------------|-----------|
| Add/Create/Post | POST      |
| Get/Find/Fetch  | GET       |
| Update/Put      | PUT       |
| Delete/Remove   | DELETE    |

## Example API Routes

Generated endpoints for the `WeatherService`:

- `GET /api/weather`
- `GET /api/weather/{id}`
- `POST /api/weather`
- `PUT /api/weather/{id}`
- `DELETE /api/weather/{id}`

## Resources

- [GitHub Repository](https://github.com/Meowv/Plus.AutoApi)
- [NuGet Package](https://www.nuget.org/packages/Plus.AutoApi)
- [Sample Project](https://github.com/Meowv/Plus.AutoApi/tree/master/samples/Plus.AutoApi.Sample)

## Key Benefits

1. **Zero Controller Boilerplate**
   Eliminate manual Controller class creation
2. **Coexistence Support**
   Works alongside traditional controllers
3. **Customizable Rules**
   Override default naming conventions through configuration options
4. **Swagger Ready**
   Full OpenAPI specification support
5. **DDD Alignment**
   Focus on domain logic rather than infrastructure code

> **Note**: This approach works best for CRUD-heavy applications. Complex endpoints may still require traditional controllers.

