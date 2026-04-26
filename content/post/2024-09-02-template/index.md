+++
author = "yuhao"
title = "ABP vNext Application Template: Quick Start Guide"
date = "2024-09-01"
description = "A ready-to-use project template for rapidly building applications with ABP vNext framework. This template comes pre-configured with essential components including Redis, Swagger UI, Autofac DI,..."
tags = [
    ".NET",
    ".NET Core",
    "abp vNext",
]
categories = [
    "Programming/Development",
]
+++
A ready-to-use project template for rapidly building applications with ABP vNext framework. This template comes pre-configured with essential components including Redis, Swagger UI, Autofac DI, Serilog logging, database migrations, JWT authentication, and multi-language support. It offers flexible database switching between MySQL, SQL Server, SQLite, and MongoDB through simple configuration changes - truly enabling out-of-the-box development for immediate business implementation.

## Quick Start

```bash
dotnet new -i AbpTemplate
dotnet new abp -n [YourProjectName]
```

**Generated Project Structure:**

```bash
[YourProjectName]
 ├── [YourProjectName].sln
 ├── LICENSE
 ├── src
 │   ├── [YourProjectName].Application
 │   ├── [YourProjectName].Application.Caching
 │   ├── [YourProjectName].Application.Contracts
 │   ├── [YourProjectName].DbMigrator
 │   ├── [YourProjectName].Domain
 │   ├── [YourProjectName].Domain.Shared
 │   ├── [YourProjectName].EntityFrameworkCore
 │   ├── [YourProjectName].EntityFrameworkCore.DbMigrations
 │   ├── [YourProjectName].HttpApi
 │   ├── [YourProjectName].HttpApi.Host
 │   └── [YourProjectName].MongoDB
 └── test
     ├── [YourProjectName].Application.Tests
     ├── [YourProjectName].Domain.Tests
     ├── [YourProjectName].EntityFrameworkCore.Tests
     ├── [YourProjectName].MongoDB.Tests
     └── [YourProjectName].TestBase
```

## NuGet Package

Available at: [https://www.nuget.org/packages/AbpTemplate](https://www.nuget.org/packages/AbpTemplate)

## Open Source Repository

GitHub: [https://github.com/Meowv/AbpTemplate](https://github.com/Meowv/AbpTemplate)

---

**Key Features:**
✅ Multi-database support with one-click configuration switching
✅ Pre-built infrastructure for caching, logging, and API documentation
✅ Modular architecture following ABP framework best practices
✅ Integrated identity management with JWT authentication
✅ Cross-cutting concern implementations for enterprise applications
✅ Unit test projects for core application layers

Simply install the template and start focusing on your business logic implementation!

