+++
author = "yuhao"
title = "Implementing API Gateway in .NET Core with Ocelot"
date = "2024-10-14"
description = "An API Gateway acts as a unified entry point for exposing APIs publicly. In .NET Core ecosystems, Ocelot provides a robust solution for implementing API gateways. Alternative libraries include..."
tags = [
    ".NET",
    ".NET Core",
    "ApiGateway",
]
categories = [
    "Programming/Development",
]
+++
## Introduction to API Gateways

An API Gateway acts as a unified entry point for exposing APIs publicly. In .NET Core ecosystems, **Ocelot** provides a robust solution for implementing API gateways. Alternative libraries include [ProxyKit](https://github.com/proxykit/ProxyKit) and Microsoft's [YARP](https://github.com/microsoft/reverse-proxy). This guide demonstrates practical implementation using Ocelot.

**Key Resources**:

- Official Docs: [https://ocelot.readthedocs.io](https://ocelot.readthedocs.io)
- GitHub: [https://github.com/ThreeMammals/Ocelot](https://github.com/ThreeMammals/Ocelot)

---

## Implementation Walkthrough

### 1. Sample API Services Setup

Create two sample API projects (`Api_A` and `Api_B`) with distinct endpoints:

**Api_A/Controllers/WeatherForecastController.cs**

```csharp
[ApiController]
[Route("api/[controller]")]
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Source = "Api_A",
            Date = DateTime.Now.AddDays(index),
            TemperatureC = Random.Shared.Next(-20, 55)
        });
    }
}
```

**Api_B/Controllers/WeatherForecastController.cs**

```csharp
[ApiController]
[Route("api/[controller]")]
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Source = "Api_B",
            Date = DateTime.Now.AddDays(index),
            TemperatureC = Random.Shared.Next(-20, 55)
        });
    }
}
```

### 2. Docker Deployment

Containerize and deploy the APIs:

```powershell
# Build images
docker build -t api_a:dev -f ./Api_A/Dockerfile .
docker build -t api_b:dev -f ./Api_B/Dockerfile .

# Run containers
docker run -d -p 5050:80 --name api_a api_a:dev
docker run -d -p 5051:80 --name api_b api_b:dev
```

---

## Gateway Configuration with Ocelot

### 1. Install NuGet Package

```powershell
Install-Package Ocelot
```

### 2. Create ocelot.json Configuration

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/WeatherForecast",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 5050 }
      ],
      "UpstreamPathTemplate": "/ApiA/WeatherForecast",
      "UpstreamHttpMethod": [ "Get" ]
    },
    {
      "DownstreamPathTemplate": "/api/ApiA/{id}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 5050 }
      ],
      "UpstreamPathTemplate": "/ApiA/{id}",
      "UpstreamHttpMethod": [ "Get", "Put", "Delete" ]
    },
    {
      "DownstreamPathTemplate": "/api/WeatherForecast",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 5051 }
      ],
      "UpstreamPathTemplate": "/ApiB/WeatherForecast",
      "UpstreamHttpMethod": [ "Get" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:7200"
  }
}
```

### 3. Program Configuration

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddJsonFile("ocelot.json");
builder.Services.AddOcelot();

var app = builder.Build();
app.UseOcelot().Wait();
app.Run();
```

---

## Testing the Gateway

### Sample Requests

```bash
# Access Api_A endpoints through gateway
curl -X GET "https://localhost:7200/ApiA/WeatherForecast"
curl -X GET "https://localhost:7200/ApiA/12345"

# Access Api_B endpoints through gateway  
curl -X GET "https://localhost:7200/ApiB/WeatherForecast"
curl -X DELETE "https://localhost:7200/ApiB/67890"
```

**Expected Responses**:

- API_A endpoints return data with `"Source": "Api_A"`
- API_B endpoints return data with `"Source": "Api_B"`

---

## Key Features Enabled

1. **Request Routing**:
   
   - `/ApiA/*` routes to API_A service
   - `/ApiB/*` routes to API_B service
2. **Protocol Translation**:
   Handles HTTP/S conversion between client and services
3. **Method Filtering**:
   Restricts endpoints to specified HTTP methods
4. **Port Abstraction**:
   Single entry point (7200) masks multiple backend ports (5050/5051)

---

## Advanced Capabilities

Ocelot supports additional enterprise features:

- **Authentication**: JWT/OAuth2 integration
- **Rate Limiting**: Prevent API abuse
- **Load Balancing**: Distribute traffic across instances
- **Circuit Breaking**: Fail-fast mechanism
- **Request Aggregation**: Combine multiple API responses

For production deployments, consider:

- Health checks integration
- Distributed configuration
- Centralized logging
- Container orchestration

---

Next Steps:
Explore [Ocelot document](https://ocelot.readthedocs.io) for advanced configuration scenarios and security best practices.

