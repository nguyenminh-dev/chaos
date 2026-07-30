# Troubleshooting Guide - Wion Platform MultiTenancy

## 🔍 Common Issues and Solutions

### Issue 1: "Tenant not found" Errors

**Symptoms:**
- 404 errors when resolving tenant
- `TenantResolutionResult` returns null
- Business logic fails with "tenant not available"

**Possible Causes:**
1. Authentication Service URL misconfigured
2. Authentication Service unavailable
3. Tenant ID doesn't exist in database
4. Network connectivity issues

**Solutions:**

#### 1. Verify Authentication Service Configuration
```bash
# Test Authentication Service endpoint directly
curl https://your-auth-service.com/api/v1/tenants/{tenant-id}

# Expected response:
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "demo-tenant",
  "isActive": true,
  "connectionString": "..."
}
```

#### 2. Check Configuration
```json
{
  "WionMultiTenancy": {
    "BaseUrl": "https://auth-service.com",  // ← Verify this is correct
    "TenantByIdPath": "/api/v1/tenants/{id}",
    "TenantByNamePath": "/api/v1/tenants/by-name/{name}"
  }
}
```

#### 3. Enable Detailed Logging
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

#### 4. Check Network Connectivity
```bash
# Test from service to authentication service
ping auth-service.com
telnet auth-service.com 443
```

---

### Issue 2: High Latency on Tenant Resolution

**Symptoms:**
- Slow API responses (>200ms)
- Frequent tenant resolution delays
- Performance degradation under load

**Possible Causes:**
1. Low cache hit rate
2. Authentication Service performance issues
3. Network latency
4. Cache configuration issues

**Solutions:**

#### 1. Monitor Cache Hit Rate
```csharp
// Add logging to monitor cache performance
Logger.LogInformation("Cache Hit Rate: {HitRate}%", 
    (cacheHits * 100.0) / totalRequests);
```

#### 2. Increase Cache Duration
```json
{
  "WionMultiTenancy": {
    "CacheDuration": "01:00:00"  // Increase from 30 minutes to 1 hour
  }
}
```

#### 3. Verify Redis Performance
```bash
# Check Redis latency
redis-cli ping
# Should respond with "PONG" in <5ms

# Check Redis memory usage
redis-cli info memory
```

#### 4. Optimize Authentication Service
- Add database indexes on tenant queries
- Implement response caching at Authentication Service level
- Use connection pooling

---

### Issue 3: Wrong Tenant Resolved

**Symptoms:**
- Business logic uses wrong tenant context
- Data leakage between tenants
- User sees incorrect data

**Possible Causes:**
1. Tenant resolution priority issues
2. Multiple tenant sources in HTTP request
3. JWT token contains incorrect tenant_id claim
4. Middleware execution order problems

**Solutions:**

#### 1. Check Resolution Priority
```csharp
// Current priority (highest to lowest):
1. JWT Token (tenant_id claim)
2. HTTP Header (X-TenantId)
3. Cookie (__tenant)
4. Query String (__tenant) - DISABLED by default
```

#### 2. Verify Tenant Sources in Request
```bash
# Check what tenant information is being sent
curl -X GET https://your-service.com/api/products \
  -H "Authorization: Bearer <jwt-token>" \
  -H "X-TenantId: 123e4567-e89b-12d3-a456-426614174000" \
  -v
```

#### 3. Enable Detailed Logging
```json
{
  "WionAspNetCoreMultiTenancy": {
    "EnableDetailedLogging": true
  }
}
```

#### 4. Check JWT Token Claims
```javascript
// Decode JWT token (use jwt.io or similar)
{
  "sub": "user-id",
  "tenant_id": "123e4567-e89b-12d3-a456-426614174000", // ← Verify this is correct
  // ... other claims
}
```

---

### Issue 4: Cache Not Working

**Symptoms:**
- Every request calls Authentication Service
- No cache hits in logs
- Performance issues

**Possible Causes:**
1. Redis connection issues
2. Cache disabled in configuration
3. Cache duration set to 0
4. Serialization issues

**Solutions:**

#### 1. Verify Redis Connection
```bash
# Test Redis connection
redis-cli -h localhost -p 6379 ping

# Check if Redis is running
docker ps | grep redis

# Check Redis logs
docker logs redis-container
```

#### 2. Check Cache Configuration
```json
{
  "WionMultiTenancy": {
    "EnableCache": true,  // ← Verify this is true
    "CacheDuration": "00:30:00"  // ← Verify duration is not 0
  },
  "Redis": {
    "Configuration": "localhost:6379"
  }
}
```

#### 3. Monitor Cache Keys
```bash
# Check what's in Redis cache
redis-cli keys "tenant:*"

# Check specific cache entry
redis-cli get "tenant:id:123e4567-e89b-12d3-a456-426614174000"
```

#### 4. Test Cache Functionality
```csharp
// Manual cache test
var tenant1 = await TenantStore.FindAsync(tenantId); // Should miss cache
var tenant2 = await TenantStore.FindAsync(tenantId); // Should hit cache
```

---

### Issue 5: Authentication Service Connection Timeouts

**Symptoms:**
- Timeout exceptions in logs
- Failed tenant resolution
- Circuit breaker triggers

**Possible Causes:**
1. Authentication Service down
2. Network issues
3. Timeout value too low
4. Authentication Service overload

**Solutions:**

#### 1. Check Authentication Service Status
```bash
# Test Authentication Service health
curl https://auth-service.com/health

# Check Authentication Service logs
docker logs auth-service-container
```

#### 2. Increase Timeout
```json
{
  "WionMultiTenancy": {
    "RequestTimeout": "00:00:30"  // Increase from 10 seconds to 30 seconds
  }
}
```

#### 3. Implement Retry Policy (Future Enhancement)
```csharp
// Not yet implemented, planned for future release
public class RetryHttpTenantProvider : IWionTenantProvider
{
    // Add retry logic with exponential backoff
}
```

#### 4. Scale Authentication Service
- Add more instances
- Implement load balancing
- Optimize database queries

---

### Issue 6: Tenant ID Format Mismatches

**Symptoms:**
- Guid parsing errors
- "Invalid tenant ID format" exceptions
- Type conversion failures

**Possible Causes:**
1. String vs Guid confusion
2. Invalid GUID format in HTTP requests
3. TenantDto serialization issues

**Solutions:**

#### 1. Verify GUID Format
```csharp
// Valid GUID format: "123e4567-e89b-12d3-a456-426614174000"
// Invalid formats: "tenant-123", "123", etc.

Guid.TryParse(tenantIdString, out var tenantId);
```

#### 2. Check HTTP Headers
```bash
# Correct format:
X-TenantId: 123e4567-e89b-12d3-a456-426614174000

# Incorrect format:
X-TenantId: tenant-123  // ❌ Not a valid GUID
```

#### 3. Verify TenantDto Structure
```csharp
// Ensure TenantDto.Id is Guid type
public class TenantDto
{
    public Guid Id { get; set; }  // ← Should be Guid, not string
    public string Name { get; set; }
}
```

---

### Issue 7: Module Registration Problems

**Symptoms:**
- Dependency injection errors
- "Service not registered" exceptions
- Application startup failures

**Possible Causes:**
1. Missing module dependencies
2. Incorrect module registration order
3. Missing service collection extensions

**Solutions:**

#### 1. Verify Module Dependencies
```csharp
[DependsOn(
    typeof(WionAspNetCoreMultiTenancyModule),  // ← Must be first
    typeof(WionAbpMultiTenancyModule),
    typeof(AbpAspNetCoreMultiTenancyModule)
)]
```

#### 2. Check Service Registration
```csharp
// Ensure provider is registered
context.Services.AddHttpTenantProvider(options =>
{
    options.BaseUrl = configuration["AuthenticationService:BaseUrl"];
});
```

#### 3. Verify Project References
```xml
<ItemGroup>
  <ProjectReference Include="..\..\frameworks\Wion.Platform.MultiTenancy\Wion.Platform.MultiTenancy.csproj" />
  <ProjectReference Include="..\..\frameworks\Wion.Platform.AspNetCore.MultiTenancy\Wion.Platform.AspNetCore.MultiTenancy.csproj" />
</ItemGroup>
```

---

## 🔧 Diagnostic Tools

### Enable Comprehensive Logging

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "Wion.Platform.MultiTenancy": "Trace",
      "Wion.Platform.AspNetCore.MultiTenancy": "Trace",
      "Volo.Abp.MultiTenancy": "Debug"
    }
  }
}
```

### Cache Statistics

```csharp
// Add to your service for monitoring
public class CacheStatistics
{
    public long CacheHits { get; set; }
    public long CacheMisses { get; set; }
    public double HitRate => CacheHits * 100.0 / (CacheHits + CacheMisses);
}
```

### Health Check Endpoint

```csharp
// Add health check for tenant resolution
app.MapGet("/health/tenant", async (ITMTCurrentTenant currentTenant) =>
{
    return new
    {
        IsAvailable = currentTenant.IsAvailable,
        TenantId = currentTenant.Id,
        TenantName = currentTenant.Name
    };
});
```

---

## 📞 Escalation Path

### Level 1: Local Troubleshooting
- Check configuration files
- Verify network connectivity
- Review application logs
- Test endpoints manually

### Level 2: Service-Level Issues
- Check Authentication Service status
- Verify Redis connectivity
- Review service dependencies
- Check resource utilization

### Level 3: Architecture Issues
- Review tenant resolution strategy
- Analyze performance patterns
- Consider caching improvements
- Evaluate service boundaries

---

**Related Documentation:**
- [Configuration Examples](configuration.md)
- [Architecture Guide](architecture.md)
- [Implementation Plan](implementation-plan.md)
