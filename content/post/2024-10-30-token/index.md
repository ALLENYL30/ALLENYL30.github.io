+++
author = "yuhao"
title = "Implementing JWT Authentication in .NET Core"
date = "2024-10-29"
description = "This guide demonstrates JWT authentication implementation in .NET Core with custom configurations and error handling."
tags = [
    ".NET",
    ".NET Core",
    "jwt",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
This guide demonstrates JWT authentication implementation in .NET Core with custom configurations and error handling.

---

## 1. NuGet Package Installation

```powershell
Install-Package Microsoft.AspNetCore.Authentication.JwtBearer
```

---

## 2. Configuration Settings

**appsettings.json**

```json
{
  "JWT": {
    "ClockSkew": 10,
    "ValidAudience": "https://meowv.com",
    "ValidIssuer": "StarPlus",
    "IssuerSigningKey": "6Zi/5pifUGx1c+... (base64 key)",
    "Expires": 30
  }
}
```

---

## 3. Service Configuration

**Startup.cs**

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options => {
            options.TokenValidationParameters = new TokenValidationParameters {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.FromSeconds(
                    Convert.ToInt32(Configuration["JWT:ClockSkew"])),
                ValidAudience = Configuration["JWT:ValidAudience"],
                ValidIssuer = Configuration["JWT:ValidIssuer"],
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(Configuration["JWT:IssuerSigningKey"]))
            };
        });

services.AddAuthorization();
```

---

## 4. Token Generation Endpoint

**AuthController.cs**

```csharp
[HttpGet("Token")]
public string GenerateToken(string username, string password)
{
    if (username == "meowv" && password == "123")
    {
        var claims = new[] {
            new Claim(ClaimTypes.Name, username),
            new Claim(ClaimTypes.Email, "123@meowv.com"),
            new Claim(JwtRegisteredClaimNames.Exp, 
                DateTimeOffset.UtcNow.AddMinutes(30).ToUnixTimeSeconds().ToString())
        };

        var credentials = new SigningCredentials(
            new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration["JWT:IssuerSigningKey"])),
            SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: Configuration["JWT:ValidIssuer"],
            audience: Configuration["JWT:ValidAudience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(30),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    throw new UnauthorizedAccessException("Invalid credentials");
}
```

**Sample Output**:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## 5. Protected Endpoints

```csharp
[Authorize]
[HttpGet("secure-data")]
public IActionResult GetSecureData() => Ok("Protected resource");

[AllowAnonymous] 
[HttpGet("public-data")]
public IActionResult GetPublicData() => Ok("Public resource");
```

**Authorization Test Results**:

- `/secure-data` without token: **401 Unauthorized**
- `/secure-data` with valid token: **200 OK**
- `/public-data`: Always accessible

---

## 6. Custom Error Handling

**Custom 401 Response**

```csharp
options.Events = new JwtBearerEvents {
    OnChallenge = async context => {
        context.HandleResponse();
        context.Response.StatusCode = 401;
        await context.Response.WriteAsync(
            JsonSerializer.Serialize(new {
                success = false,
                message = "Authentication required"
            }));
    }
};
```

---

## 7. Alternative Token Locations

**Query Parameter Support**

```csharp
options.Events = new JwtBearerEvents {
    OnMessageReceived = context => {
        context.Token = context.Request.Query["access_token"];
        return Task.CompletedTask;
    }
};
```

**Usage**:
`GET /api/data?access_token=eyJhbGciOiJIUzI1NiIs...`

---

## Key Features

- Configurable expiration and clock skew
- Customizable error responses
- Multiple token location support (Header/Query/Cookie)
- Claims-based authorization
- Standard-compliant JWT validation

Always store signing keys securely and rotate them periodically for production systems.

