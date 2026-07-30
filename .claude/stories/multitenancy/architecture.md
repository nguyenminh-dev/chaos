# Wion Platform MultiTenancy - Architecture Guide

## 🎯 Architecture Overview

This module implements a **centralized tenant management pattern** where the Authentication Service owns tenant data and other services resolve tenant information via HTTP/gRPC calls.

## 📊 System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Authentication Service                         │
│  (Tenant Data Owner)                                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Database                                                 │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │  AbpTenants Table                                    │ │  │
│  │  │  - Id (Guid)                                         │ │  │
│  │  │  - Name                                              │ │  │
│  │  │  - ConnectionString                                  │ │  │
│  │  │  - IsActive                                          │ │  │
│  │  │  - ExtraProperties                                    │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  DatabaseTenantProvider (Database-Based)                  │  │
│  │  - Implements IWionTenantProvider                         │  │
│  │  - Queries database directly                             │  │
│  │  - Used by Authentication Service                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Tenant HTTP/gRPC Endpoints                              │  │
│  │  - GET /api/v1/tenants/{id}                              │  │
│  │  - GET /api/v1/tenants/by-name/{name}                    │  │
│  └───────────────▲──────────────────────────────────────────┘  │
└───────────────────┼─────────────────────────────────────────────┘
                    │ HTTP/gRPC
                    │
                    │ Called by other services
                    │
┌───────────────────┴─────────────────────────────────────────────┐
│              Other Services (Product, Order, etc.)              │
│                  (No Tenant Database)                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  WionTenantStore (Cache + Provider Coordination)         │  │
│  │  - Implements ITenantStore                                │  │
│  │  - Check Redis cache first                               │  │
│  │  - Call HttpWionTenantProvider if cache miss              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HttpWionTenantProvider                                   │  │
│  │  - Makes HTTP calls to Auth Service                      │  │
│  │  - Handles retries and timeouts                          │  │
│  │  - Maps JSON to TenantConfiguration                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Distributed Cache (Redis)                                │  │
│  │  - Key: tenant:id:{id}                                   │  │
│  │  - Key: tenant:name:{name}                               │  │
│  │  - Duration: 30 minutes                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 🔄 Complete Data Flow

### Scenario 1: Other Service Receives Request with Tenant Context

```
1. HTTP Request arrives at Product Service
   ├─ Headers: X-TenantId: 123e4567-e89b-12d3-a456-426614174000
   └─ Cookies: __tenant=123e4567-e89b-12d3-a456-426614174000

2. WionMultiTenancyMiddleware intercepts request
   ├─ Extracts tenant identifier from multiple sources
   └─ Calls ITMTTenantConfigurationProvider.GetAsync()

3. WionTenantConfigurationProvider
   ├─ Checks JWT token → finds tenant_id claim
   ├─ Falls back to HTTP header (X-TenantId)
   ├─ Falls back to Cookie (__tenant)
   └─ Returns tenant identifier: "123e4567-e89b-12d3-a456-426614174000"

4. WionTenantStore receives tenant ID
   ├─ Check Redis cache: tenant:id:123e4567-e89b-12d3-a456-426614174000
   │  └─ Cache MISS
   ├─ Call HttpWionTenantProvider.FindAsync(Guid id)
   └─ Provider calls Authentication Service

5. Authentication Service processes request
   ├─ HTTP GET /api/v1/tenants/123e4567-e89b-12d3-a456-426614174000
   ├─ DatabaseTenantProvider queries database
   ├─ Returns tenant info as JSON
   └─ Response: { id, name, connectionString, isActive }

6. Product Service processes response
   ├─ HttpWionTenantProvider deserializes JSON
   ├─ Maps to TenantConfiguration
   ├─ WionTenantStore caches in Redis (30 minutes)
   └─ Returns to ABP pipeline

7. ABP Pipeline completes tenant setup
   ├─ Sets ITMTCurrentTenant with tenant info
   ├─ DbContext filters by tenant
   └─ Business logic executes with tenant context
```

### Scenario 2: Cache Hit (Subsequent Requests)

```
1. HTTP Request arrives at Product Service
   └─ Headers: X-TenantId: 123e4567-e89b-12d3-a456-426614174000

2. WionTenantStore
   ├─ Check Redis cache: tenant:id:123e4567-e89b-12d3-a456-426614174000
   └─ Cache HIT! Returns immediately

3. No HTTP call to Authentication Service
4. Faster response time (~1-2ms vs ~50-200ms)
```

## 🏗️ Component Responsibilities

### Authentication Service (tpos-authentication)

**Responsibilities:**
- ✅ Own and manage tenant database
- ✅ Provide tenant CRUD operations
- ✅ Expose tenant information via HTTP/gRPC APIs
- ✅ Use `DatabaseTenantProvider` (database-based)
- ✅ Maintain tenant data consistency

**Should NOT:**
- ❌ Use `WionTenantStore` (that's for OTHER services)
- ❌ Make HTTP calls to itself

**Technology Stack:**
- Database: PostgreSQL with `AbpTenants` table
- Provider: `DatabaseTenantProvider`
- Framework: ABP Tenant Management

### Other Services (Product, Order, etc.)

**Responsibilities:**
- ✅ Use `WionTenantStore` (cache + provider coordination)
- ✅ Call Authentication Service for tenant info
- ✅ Cache tenant information locally (Redis)
- ✅ Handle Authentication Service unavailability gracefully

**Should NOT:**
- ❌ Have their own tenant database
- ❌ Use database-based tenant providers
- ❌ Manage tenant CRUD operations

**Technology Stack:**
- Cache: Redis (distributed cache)
- Provider: `HttpWionTenantProvider`
- Framework: Wion.Platform.MultiTenancy modules

## 📋 Service Classification Matrix

| Service | Has Tenant DB? | TenantStore Implementation | Provider Type | Uses Wion Modules? |
|---------|----------------|----------------------------|---------------|---------------------|
| **Authentication** | ✅ Yes | `DatabaseTenantProvider` | Database | ❌ No |
| **Product** | ❌ No | `WionTenantStore` | HTTP | ✅ Yes |
| **Order** | ❌ No | `WionTenantStore` | HTTP | ✅ Yes |
| **Inventory** | ❌ No | `WionTenantStore` | HTTP | ✅ Yes |
| **Payment** | ❌ No | `WionTenantStore` | HTTP | ✅ Yes |

## 🔌 Integration Points

### Authentication Service API Contract

**GET `/api/v1/tenants/{id}`**
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "demo-tenant",
  "isActive": true,
  "connectionString": "Server=localhost;Database=DemoDb;",
  "extraProperties": {
    "editionId": "premium",
    "customField": "value"
  }
}
```

**GET `/api/v1/tenants/by-name/{name}`**
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "demo-tenant",
  "isActive": true,
  "connectionString": "Server=localhost;Database=DemoDb;",
  "extraProperties": {}
}
```

### Module Dependencies

```
Wion.Platform.MultiTenancy
├── TMT.Abp.MultiTenancy (core infrastructure)
├── AbpMultiTenancyModule
├── AbpCachingModule
└── AbpHttpClientModule

Wion.Platform.AspNetCore.MultiTenancy
├── TMT.Abp.MultiTenancy (core infrastructure)
├── Wion.Platform.MultiTenancy
├── AbpAspNetCoreMultiTenancyModule
└── TMTAbpMultiTenancyModule
```

## ⚖️ Architectural Trade-offs

### Advantages

**For Other Services:**
- 🎯 **Simpler Architecture**: No tenant database management
- 🎯 **Consistency**: All services use same tenant data
- 🎯 **Faster Development**: Less database setup per service
- 🎯 **Flexibility**: Easy to add new services without tenant DB

**For Authentication Service:**
- 🎯 **Data Ownership**: Remains the source of truth
- 🎯 **Control**: Can manage tenants independently
- 🎯 **Security**: Centralized tenant management

**Overall System:**
- 🎯 **Scalability**: Authentication Service can be optimized independently
- 🎯 **Maintainability**: Single place to update tenant logic
- 🎯 **Performance**: Caching reduces load on Authentication Service

### Considerations

**Network Dependency:**
- ⚠️ Other services depend on Authentication Service availability
- ✅ Mitigated by caching (30 minutes)
- ✅ Mitigated by retry policies and circuit breakers

**Latency:**
- ⚠️ HTTP calls add latency (cache miss scenarios)
- ✅ Acceptable for most operations (~50-200ms)
- ✅ Cache reduces this significantly (~1-2ms)

**Complexity:**
- ⚠️ Network configuration required
- ✅ Simpler than managing tenant databases per service

## 🔧 Configuration Examples

### Authentication Service (`appsettings.json`)
```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=AuthDb;"
  },
  "MultiTenancy": {
    "IsEnabled": true
  }
}
```

### Other Services (`appsettings.json`)
```json
{
  "WionMultiTenancy": {
    "BaseUrl": "https://auth-service.com",
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
    "EnableDetailedLogging": true
  }
}
```

## 📊 Performance Characteristics

### Cache Performance
- **Cache Hit**: ~1-2ms (Redis lookup)
- **Cache Miss**: ~50-200ms (HTTP call + Redis write)
- **Database Query** (Auth Service): ~5-20ms

### Target Metrics
- **Cache Hit Rate**: >90% (normal operations)
- **Acceptable**: 70-90% cache hit rate
- **Poor**: <70% cache hit rate (increase cache duration)

### Scalability Factors
- **Authentication Service**: Handles all tenant queries
- **Other Services**: Minimal load, mostly cache hits
- **Redis**: Handles cache storage and retrieval

## 🔍 Monitoring & Debugging

### Key Metrics to Monitor
- Cache hit/miss ratio
- HTTP call latency to Authentication Service
- Error rates from Authentication Service
- Tenant resolution frequency

### Logging Configuration
```json
{
  "Logging": {
    "LogLevel": {
      "Wion.Platform.MultiTenancy": "Debug",
      "Wion.Platform.AspNetCore.MultiTenancy": "Debug"
    }
  }
}
```

---

**Related Documentation:**
- [Quick Start Guide](quick-start.md)
- [Troubleshooting Guide](troubleshooting.md)
- [Implementation Plan](implementation-plan.md)
