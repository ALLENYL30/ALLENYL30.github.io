+++
author = "yuhao"
title = "Docker Compose Setup for Consul Cluster"
date = "2024-11-18"
description = "Deploy a 3-server-node cluster + 1 client node using Docker:"
tags = [
    ".NET",
    ".NET Core",
    "Consul",
]
categories = [
    "Programming/Development",
]
+++
Deploy a 3-server-node cluster + 1 client node using Docker:

```yaml
version: "3"

services:
service_1:
image: consul
command: agent -server -client=0.0.0.0 -bootstrap-expect=3 -node=service_1
volumes:
- /data/service_1:/data
service_2:
image: consul
command: agent -server -client=0.0.0.0 -retry-join=service_1 -node=service_2
depends_on: [service_1]
service_3:
image: consul
command: agent -server -client=0.0.0.0 -retry-join=service_1 -node=service_3
depends_on: [service_1]
client_1:
image: consul
command: agent -client=0.0.0.0 -retry-join=service_1 -ui -node=client_1
ports: ["8500:8500"]
depends_on: [service_2, service_3]

```
Key components:

- 3 server nodes for cluster consensus
- 1 client node with web UI (accessible via `http://localhost:8500`)
- Persistent volume mapping for data storage

---

## 🛠️ Service Registration Implementation

### 1. Configuration (appsettings.json)

```json
{
  "Consul": {
    "Address": "http://host.docker.internal:8500",
    "HealthCheck": "/healthcheck",
    "Name": "ServiceA",
    "Port": "5050"
  }
}
```

### 2. Consul Registration Extension

```csharp
public static class ConsulExtensions
{
    public static IApplicationBuilder UseConsul(
        this IApplicationBuilder app, 
        IConfiguration config, 
        IHostApplicationLifetime lifetime)
    {
        var client = new ConsulClient(c => c.Address = new Uri(config["Consul:Address"]));

        var registration = new AgentServiceRegistration
        {
            ID = Guid.NewGuid().ToString(),
            Name = config["Consul:Name"],
            Address = config["Consul:Ip"],
            Port = int.Parse(config["Consul:Port"]),
            Check = new AgentServiceCheck
            {
                Interval = TimeSpan.FromSeconds(10),
                HTTP = $"http://{config["Consul:Ip"]}:{config["Consul:Port"]}{config["Consul:HealthCheck"]}",
                Timeout = TimeSpan.FromSeconds(5)
            }
        };

        client.Agent.ServiceRegister(registration).Wait();
        lifetime.ApplicationStopping.Register(() => 
            client.Agent.ServiceDeregister(registration.ID).Wait());

        return app;
    }
}
```

### 3. Health Check Endpoint

```csharp
[ApiController]
[Route("[controller]")]
public class HealthCheckController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok();
}
```

### 4. Service Endpoint Example

```csharp
[ApiController]
[Route("api/[controller]")]
public class ServiceAController : ControllerBase
{
    [HttpGet]
    public IActionResult Get([FromServices] IConfiguration config)
    {
        return Ok(new {
            Service = nameof(ServiceA),
            Timestamp = DateTime.Now.ToString("G"),
            Port = config["Consul:Port"]
        });
    }
}
```

---

## 🔍 Service Discovery Implementation

### 1. Service Discovery Client

```csharp
public class ServiceDiscoveryClient
{
    private readonly ConsulClient _consul;
    private ConcurrentBag<string> _serviceAUrls = new();
    private ConcurrentBag<string> _serviceBUrls = new();

    public ServiceDiscoveryClient(IConfiguration config)
    {
        _consul = new ConsulClient(c => 
            c.Address = new Uri(config["Consul:Address"]));
    }

    public async Task<string> GetServiceA()
    {
        var client = new HttpClient();
        var endpoint = _serviceAUrls.ElementAt(
            new Random().Next(_serviceAUrls.Count));
        return await client.GetStringAsync($"{endpoint}/api/servicea");
    }

    public void InitializeServices()
    {
        Task.Run(async () => 
        {
            var query = new QueryOptions { WaitTime = TimeSpan.FromMinutes(5) };
            while(true) 
            {
                await UpdateServices("ServiceA", query, _serviceAUrls);
                await UpdateServices("ServiceB", query, _serviceBUrls);
            }
        });
    }

    private async Task UpdateServices(string serviceName, QueryOptions query, 
        ConcurrentBag<string> target)
    {
        var response = await _consul.Health.Service(serviceName, null, true, query);
        if (query.WaitIndex != response.LastIndex)
        {
            query.WaitIndex = response.LastIndex;
            target.Clear();
            foreach(var service in response.Response)
                target.Add($"http://{service.Service.Address}:{service.Service.Port}");
        }
    }
}
```

### 2. Client Controller

```csharp
[ApiController]
[Route("api")]
public class GatewayController : ControllerBase
{
    private readonly ServiceDiscoveryClient _client;

    public GatewayController(ServiceDiscoveryClient client) => _client = client;

    [HttpGet("services")]
    public async Task<IActionResult> GetServices()
    {
        return Ok(new {
            ServiceA = await _client.GetServiceA(),
            ServiceB = await _client.GetServiceB()
        });
    }
}
```

---

## 🚀 Deployment Workflow

1. Build Docker images:

```bash
docker build -t service_a:latest -f ServiceA/Dockerfile .
docker build -t service_b:latest -f ServiceB/Dockerfile .
```

2. Start service instances:

```bash
docker run -d -p 5050:80 service_a:latest --Consul:Port=5050
docker run -d -p 5051:80 service_a:latest --Consul:Port=5051
docker run -d -p 5060:80 service_b:latest --Consul:Port=5060
```

3. Verify registration in Consul UI:

```
Registered Services:
- ServiceA (3 healthy instances)
- ServiceB (2 healthy instances)
```

---

Key Features Demonstrated:

- Cluster setup with automatic node discovery
- Health monitoring with configurable intervals
- Dynamic service endpoint registration/deregistration
- Load balancing through random instance selection
- HTTP client integration for service consumption
- Dockerized deployment workflow

> **Note**: The `host.docker.internal` is used for cross-container communication in Docker environments. Replace with `localhost` for local non-Docker development.
> 
> ---

**Official Resources**:

- Consul Official Website**: [https://www.consul.io](https://www.consul.io)
- .NET Client GitHub**: [https://github.com/G-Research/consuldotnet](https://github.com/G-Research/consuldotnet)

---

