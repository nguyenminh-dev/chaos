# Wion Platform MultiTenancy - Quick Start Guide

## 🎯 Objective

Implement centralized tenant management where **Authentication Service** owns tenant data and **other services** resolve tenant information via HTTP calls.

## 📋 Prerequisites

- ✅ Authentication Service (`tpos-authentication`) with tenant database
- ✅ Redis cache available for distributed caching
- ✅ HTTP/gRPC endpoints exposed by Authentication Service
- ✅ Other services need to resolve tenant context

## 🚀 Quick Start

### Step 1: Authentication Service Setup (Already Done ✅)

Your Authentication Service should already have:

```csharp
// In TPos.Service.Delegates
[Dependency(ReplaceServices = true)]
[ExposeServices(typeof(ITMTTenantStore))]
public class WiOnPosTenantStore : TenantStore, ITransientDependency
{
    // Database-based implementation
}
```

**Verify endpoints are exposed:**
```bash
# Test Authentication Service tenant endpoints
GET https://your-auth-service.com/api/v1/tenants/{id}
GET https://your-auth-service.com/api/v1/tenants/by-name/{name}
```

### Step 2: Add Module to Other Services

For each service that needs tenant resolution:

#### 2.1 Add Project Reference

```xml
<!-- In YourService.csproj -->
<ItemGroup>
  <ProjectReference Include="..\..\frameworks\Wion.Platform.MultiTenancy\Wion.Platform.MultiTenancy.csproj" />
</ItemGroup>
```

#### 2.2 Add Module Dependency

```csharp
// In YourServiceModule.cs
[DependsOn(
    typeof(WionAbpMultiTenancyModule),  // Add this
    typeof(AbpAspNetCoreMultiTenancyModule),
    // ... other dependencies
)]
public class YourServiceModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // ... existing configuration
    }
}
```

#### 2.3 Configure Authentication Service URL

In `appsettings.json`:

```json
{
  "WionMultiTenancy": {
    "BaseUrl": "https://your-auth-service.com",
    "CacheDuration": "00:30:00",
    "EnableCache": true,
    "RequestTimeout": "00:00:10"
  }
}
```

## 📚 Additional Documentation

- [Architecture Guide](architecture-guide.md) - Detailed architecture explanation
- [Wion.Platform.MultiTenancy README](../../src/frameworks/Wion.Platform.MultiTenancy/README.md) - Module documentation
- [ABP Multi-Tenancy](https://docs.abp.io/en/abp/latest/Multi-Tenancy) - ABP framework documentation