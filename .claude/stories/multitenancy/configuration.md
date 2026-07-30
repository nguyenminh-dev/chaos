# Configuration Examples - Wion Platform MultiTenancy

## 🎯 Overview

This guide provides practical configuration examples for different service types and deployment scenarios.

## 📋 Configuration Files

### Authentication Service Configuration

The Authentication Service uses standard ABP multi-tenancy configuration since it owns the tenant database.

#### `appsettings.json`
```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=TposAuthentication;Port=5432;User Id=postgres;Password=your_password;"
  },
  "MultiTenancy": {
    "IsEnabled": true
  },
  "AuthServer": {
    "Authority": "https://auth-service.com",
    "RequireHttpsMetadata": false
  }
}
```

#### `appsettings.Production.json`
```json
{
  "ConnectionStrings": {
    "Default": "Server=prod-db-server;Database=TposAuthentication_Prod;Port=5432;User Id=app_user;Password=production_password;"
  },
  "AuthServer": {
    "Authority": "https://auth.yourcompany.com",
    "RequireHttpsMetadata": true
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning",
      "Volo.Abp": "Warning"
    }
  }
}
```

### Other Services Configuration

Services without tenant databases use the Wion.Platform.MultiTenancy modules.

#### `appsettings.json`
```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=TposProduct;Port=5432;User Id=postgres;Password=your_password;"
  },
  "WionMultiTenancy": {
    "BaseUrl": "https://auth-service.com",
    "TenantByIdPath": "/api/v1/tenants/{id}",
    "TenantByNamePath": "/api/v1/tenants/by-name/{name}",
    "CacheDuration": "00:30:00",
    "EnableCache": true,
    "RequestTimeout": "00:00:10"
  },
  "WionAspNetCoreMultiTenancy": {
    "ResolveFromJwtToken": true,
    "ResolveFromHeader": true,
    "ResolveFromCookie": true,
    "ResolveFromQueryString": false,
    "TenantHeaderName": "X-TenantId",
    "TenantCookieName": "__tenant",
    "TenantQueryStringName": "__tenant",
    "EnableDetailedLogging": true,
    "OnTenantNotFoundError": "Return404"
  },
  "Redis": {
    "Configuration": "localhost:6379"
  }
}
```

#### `appsettings.Production.json`
```json
{
  "ConnectionStrings": {
    "Default": "Server=prod-db-server;Database=TposProduct_Prod;Port=5432;User Id=app_user;Password=production_password;"
  },
  "WionMultiTenancy": {
    "BaseUrl": "https://auth.yourcompany.com",
    "CacheDuration": "01:00:00",
    "EnableCache": true,
    "RequestTimeout": "00:00:15"
  },
  "WionAspNetCoreMultiTenancy": {
    "EnableDetailedLogging": false,
    "OnTenantNotFoundError": "Return401"
  },
  "Redis": {
    "Configuration": "prod-redis:6379,password=redis_password,ssl=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning",
      "Volo.Abp": "Warning",
      "Wion.Platform.MultiTenancy": "Information"
    }
  }
}
```

## 🔧 Module Registration

### Authentication Service Module

```csharp
// AuthenDomainModule.cs
using Authen.MultiTenancy;
using Wion.Platform.MultiTenancy;

[DependsOn(
    typeof(AbpTenantManagementDomainModule),
    typeof(WionAbpMultiTenancyModule)
)]
public class AuthenDomainModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Configure standard ABP multi-tenancy
        Configure<AbpMultiTenancyOptions>(options =>
        {
            options.IsEnabled = MultiTenancyConsts.IsEnabled;
        });

        // Configure Wion Platform MultiTenancy - use Database Provider
        context.Services.AddDatabaseTenantProvider();
    }
}
```

### Other Services Module Registration

```csharp
// ProductServiceModule.cs
using Wion.Platform.MultiTenancy;
using Wion.Platform.AspNetCore.MultiTenancy;

[DependsOn(
    typeof(WionAspNetCoreMultiTenancyModule),
    typeof(WionAbpMultiTenancyModule),
    typeof(AbpAspNetCoreMultiTenancyModule)
)]
public class ProductServiceModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();

        // Configure HTTP-based tenant store
        Configure<WionMultiTenancyOptions>(options =>
        {
            options.BaseUrl = configuration["AuthenticationService:BaseUrl"];
            options.TenantByIdPath = "/api/v1/tenants/{id}";
            options.TenantByNamePath = "/api/v1/tenants/by-name/{name}";
            options.CacheDuration = TimeSpan.FromMinutes(30);
            options.EnableCache = true;
            options.RequestTimeout = TimeSpan.FromSeconds(10);
        });

        // Configure tenant resolution
        Configure<WionAspNetCoreMultiTenancyOptions>(options =>
        {
            options.ResolveFromJwtToken = true;
            options.ResolveFromHeader = true;
            options.ResolveFromCookie = true;
            options.ResolveFromQueryString = false; // Security best practice
            options.TenantHeaderName = "X-TenantId";
            options.EnableDetailedLogging = true;
        });
    }
}
```

### HTTP Client Registration

```csharp
// HttpApiHostModule.cs
public override void ConfigureServices(ServiceConfigurationContext context)
{
    var configuration = context.Services.GetConfiguration();
    
    // Register HTTP client for tenant provider
    context.Services.AddHttpClient("WionTenantService", client =>
    {
        client.BaseAddress = new Uri(configuration["AuthenticationService:BaseUrl"]);
        client.Timeout = TimeSpan.FromSeconds(10);
    });
}
```

## 🔧 Middleware Configuration

### ASP.NET Core Pipeline Setup

```csharp
// HttpApiHostModule.cs
public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var app = context.GetApplicationBuilder();
    
    // ... other middleware
    
    app.UseWionMultiTenancy();  // ← Use Wion middleware instead of ABP's
    
    app.UseAuthentication();
    app.UseAuthorization();
    
    app.UseAbpRequestLocalization();
    
    // ... rest of middleware
}
```

## 🧪 Development Configuration

### Local Development Environment

```json
{
  "WionMultiTenancy": {
    "BaseUrl": "https://localhost:5001",
    "CacheDuration": "00:05:00",
    "EnableCache": true,
    "RequestTimeout": "00:00:30"
  },
  "WionAspNetCoreMultiTenancy": {
    "ResolveFromQueryString": true,
    "EnableDetailedLogging": true
  },
  "Logging": {
    "LogLevel": {
      "Wion.Platform.MultiTenancy": "Debug",
      "Wion.Platform.AspNetCore.MultiTenancy": "Debug"
    }
  }
}
```

### Docker Compose Configuration

```yaml
version: '3.8'

services:
  auth-service:
    image: tpos-auth-service:latest
    environment:
      - ConnectionStrings__Default=Server=postgres;Database=TposAuthentication;Port=5432;User Id=postgres;Password=password
      - MultiTenancy__IsEnabled=true
    ports:
      - "5001:5001"

  product-service:
    image: tpos-product-service:latest
    environment:
      - ConnectionStrings__Default=Server=postgres;Database=TposProduct;Port=5432;User Id=postgres;Password=password
      - WionMultiTenancy__BaseUrl=http://auth-service:5001
      - WionMultiTenancy__EnableCache=true
      - Redis__Configuration=redis:6379
    depends_on:
      - auth-service
      - redis

  redis:
    image: redis:latest
    ports:
      - "6379:6379"
```

## 🔧 Environment-Specific Configuration

### Development Environment
- ✅ QueryString resolution enabled for testing
- ✅ Detailed logging enabled
- ✅ Short cache duration (5 minutes)
- ✅ Lenient timeouts (30 seconds)

### Staging Environment
- ⚠️ QueryString resolution disabled
- ⚠️ Information-level logging
- ⚠️ Medium cache duration (15 minutes)
- ⚠️ Standard timeouts (10 seconds)

### Production Environment
- ❌ QueryString resolution disabled
- ❌ Warning-level logging only
- ✅ Long cache duration (60 minutes)
- ✅ Strict timeouts (15 seconds)
- ✅ HTTPS required

## 🧪 Testing Configuration

### Unit Testing with Mock Provider

```csharp
// Test setup
public class TenantServiceTests
{
    private readonly Mock<IWionTenantProvider> _mockProvider;
    
    public TenantServiceTests()
    {
        _mockProvider = new Mock<IWionTenantProvider>();
        _mockProvider.Setup(x => x.FindAsync(It.IsAny<Guid>()))
            .ReturnsAsync(new TenantConfiguration 
            { 
                Id = Guid.Parse("123e4567-e89b-12d3-a456-426614174000"),
                Name = "test-tenant" 
            });
    }
}
```

### Integration Testing Configuration

```json
{
  "WionMultiTenancy": {
    "BaseUrl": "http://localhost:5001",
    "EnableCache": false,
    "RequestTimeout": "00:00:30"
  },
  "WionAspNetCoreMultiTenancy": {
    "ResolveFromQueryString": true,
    "EnableDetailedLogging": true
  }
}
```

## 🔧 Troubleshooting Configuration

### Enable Diagnostic Logging

```json
{
  "Logging": {
    "LogLevel": {
      "Wion.Platform.MultiTenancy": "Trace",
      "Wion.Platform.AspNetCore.MultiTenancy": "Trace"
    }
  }
}
```

### Disable Caching for Testing

```json
{
  "WionMultiTenancy": {
    "EnableCache": false
  }
}
```

### Extend Timeouts for Slow Networks

```json
{
  "WionMultiTenancy": {
    "RequestTimeout": "00:01:00"
  }
}
```

---

**Related Documentation:**
- [Quick Start Guide](quick-start.md)
- [Architecture Guide](architecture.md)
- [Troubleshooting Guide](troubleshooting.md)
