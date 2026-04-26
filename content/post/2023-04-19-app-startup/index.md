+++
author = "yuhao"
title = "Deep Dive into .NET 8 WebApplication: Architecture and Components"
date = "2023-04-19"
description = "WebApplication is used to configure the HTTP pipeline and routes in web applications. In this article, I'll break down its components and structure."
tags = [
    ".NET",
    ".NET Core",
    "C#",
    "webapplication",
]
categories = [
    "Programming/Development",
]
+++
WebApplication is used to configure the HTTP pipeline and routes in web applications. In this article, I'll break down its components and structure.

```csharp
/// <summary>
/// The web application used to configure the HTTP pipeline, and routes.
/// </summary>
[DebuggerDisplay("{DebuggerToString(),nq}")]
[DebuggerTypeProxy(typeof(WebApplication.WebApplicationDebugView))]
public sealed class WebApplication : IHost, IDisposable, IApplicationBuilder, IEndpointRouteBuilder, IAsyncDisposable

```
## IHost

First and foremost, a web application is a program, and `IHost` is the abstraction of that program.

```csharp
public interface IHost : IDisposable
{
  IServiceProvider Services { get; }
  Task StartAsync(CancellationToken cancellationToken = default(CancellationToken));
  Task StopAsync(CancellationToken cancellationToken = default(CancellationToken));
}
```

It makes sense that a program has start and stop lifecycle methods. I want to highlight `IServiceProvider`, which is crucial and will be explained in detail in the dependency injection chapter. For now, just know that it's a service provider that allows you to access services, provided they're registered in the IoC container.

The code flow for `Host.StartAsync` is as follows:

```csharp
await host._hostLifetime.WaitForStartAsync(token1).ConfigureAwait(false); // Register start program
host.Services.GetService<IStartupValidator>()?.Validate(); // Validate
IHostedLifecycleService.StartingAsync
IHostedService.StartAsync
IHostedLifecycleService.StartedAsync
host._applicationLifetime.NotifyStarted();
```

Similarly, `StopAsync` follows this flow:

```csharp
IHostedLifecycleService.StoppingAsync
IHostedService.StopAsync
IHostedLifecycleService.StoppedAsync
this._logger.StoppedWithException((Exception) ex);
```

It's worth noting that `IStartupValidator`, `IHostedService`, and `IHostedLifecycleService` provide different hooks for us to inject our custom business logic by registering them with the container.

## IApplicationBuilder

WebApplication implements `IApplicationBuilder`, which provides the pipeline mechanism.

```csharp
IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware);
RequestDelegate Build();
```

Let me explain the pipeline concept: pipelines are a widespread concept in .NET, with aspect-oriented programming at their core. We'll cover this in a dedicated chapter later. For now, just know that it follows an onion model.

Additionally, as the name suggests, `IApplicationBuilder` implements the builder pattern, so WebApplication provides a `Build` method:

```csharp
public WebApplication Build()
{
  this._hostApplicationBuilder.Services.Add(this._genericWebHostServiceDescriptor);
  this.Host.ApplyServiceProviderFactory(this._hostApplicationBuilder);
  this._builtApplication = new WebApplication(this._hostApplicationBuilder.Build());
  return this._builtApplication;
}
```

Due to space constraints, I won't delve into all the details, but four hooks are executed in the Build method:

```csharp
public IHostBuilder ConfigureHostConfiguration(Action<IConfigurationBuilder> configureDelegate)
public IHostBuilder ConfigureAppConfiguration(Action<HostBuilderContext, IConfigurationBuilder> configureDelegate)
public IHostBuilder ConfigureServices(Action<HostBuilderContext, IServiceCollection> configureDelegate)
public IHostBuilder ConfigureContainer<TContainerBuilder>(Action<HostBuilderContext, TContainerBuilder> configureDelegate)
```

Ultimately, the service container is marked as read-only.

## IEndpointRouteBuilder

`IEndpointRouteBuilder` defines the convention for building routes in the application.

```csharp
ICollection<EndpointDataSource> DataSources { get; }
```

Here's the implementation in WebApplication, showing how `EndpointDataSource` implementations are composed together:

```csharp
public IReadOnlyList<Endpoint> Endpoints
{
  get
  {
    EndpointDataSource requiredService = this._webApplication.Services.GetRequiredService<EndpointDataSource>();
    return requiredService is CompositeEndpointDataSource endpointDataSource && endpointDataSource.DataSources.Intersect<EndpointDataSource>((IEnumerable<EndpointDataSource>) this._webApplication.DataSources).Count<EndpointDataSource>() != this._webApplication.DataSources.Count ? new CompositeEndpointDataSource((IEnumerable<EndpointDataSource>) this._webApplication.DataSources).Endpoints : requiredService.Endpoints;
  }
}
```

An `EndpointDataSource` is essentially a collection of Endpoints, and Endpoint is an extremely important concept in AspNetCore that we'll explore later. For now, understand that it represents a terminal point for handling HTTP requests, containing both the logic to process requests and associated metadata.

## IAsyncDisposable

`IAsyncDisposable` is an interface introduced in .NET Core 2.0 for asynchronously releasing resources. It's the asynchronous version of the `IDisposable` interface. Releasing certain resources may involve asynchronous operations, and using the synchronous release pattern of the `IDisposable` interface could block threads, affecting application performance and responsiveness.

```csharp
public interface IAsyncDisposable
{
  ValueTask DisposeAsync();
}
```

After using the object, you can use the `await using` syntactic sugar or directly call the `DisposeAsync()` method to release resources.

## Run

The `Run` function is WebApplication's start button. You can pass a URL to add it to the listening list:

```csharp
public void Run([StringSyntax("Uri")] string? url = null)
{
  this.Listen(url);
  // public static void Run(this IHost host) => host.RunAsync().GetAwaiter().GetResult();
  HostingAbstractionsHostExtensions.Run(this);
}
```

The source code for `HostingAbstractionsHostExtensions.Run` is relatively simple. The host's `StartAsync` and `WaitForShutdownAsync` were discussed above, and WebApplication's `DisposeAsync` is triggered in the finally block:

```csharp
public static async Task RunAsync(this IHost host, CancellationToken token = default(CancellationToken))
{
  try
  {
    ConfiguredTaskAwaitable configuredTaskAwaitable = host.StartAsync(token).ConfigureAwait(false);
    await configuredTaskAwaitable;
    configuredTaskAwaitable = host.WaitForShutdownAsync(token).ConfigureAwait(false);
    await configuredTaskAwaitable;
  }
  finally
  {
    if (host is IAsyncDisposable asyncDisposable)
      await asyncDisposable.DisposeAsync().ConfigureAwait(false);
    else
      host.Dispose();
  }
}
```

An interesting point here is: why trigger in the finally block? This is because the Host's lifecycle functions all use a combined cancellation token. This is a safe cancellation cooperation pattern - after the token is canceled, an `OperationCanceledException` exception is triggered, allowing normal disposal work to continue even in this situation. This is an excellent programming practice.

