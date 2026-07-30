# T-PayGate Resiliency Patterns

## Overview
Resiliency patterns for T-PayGate integration including retry mechanisms, circuit breakers, timeouts, bulkheads, and fallback strategies.

---

## 1. Retry Patterns

### 1.1 HTTP Client Retry Policy

```csharp
// Retry policy for T-PayGate API calls
private IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return Policy
        .Handle<HttpRequestException>()
        .OrResult<HttpResponseMessage>(r => 
            r.StatusCode == HttpStatusCode.ServiceUnavailable ||  // 503
            r.StatusCode == HttpStatusCode.GatewayTimeout ||      // 504
            r.StatusCode == HttpStatusCode.TooManyRequests)       // 429
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt => 
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)), // 1s, 2s, 4s
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                _logger.LogWarning(
                    "Retry {RetryCount} after {Delay}s due to: {Message}", 
                    retryCount, 
                    timespan.TotalSeconds, 
                    outcome.Exception?.Message ?? outcome.Result.StatusCode.ToString()
                );
            }
        );
}
```

### 1.2 Token Refresh Retry Policy

```csharp
// Retry policy specifically for OAuth token refresh
private IAsyncPolicy<TpgAccessToken> GetTokenRefreshPolicy()
{
    return Policy
        .Handle<ApiException>(ex => ex.StatusCode == HttpStatusCode.ServiceUnavailable)
        .Or<TimeoutException>()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(1 + retryAttempt), // 1s, 2s, 3s
            onRetry: (exception, delay, retryCount, context) =>
            {
                _logger.LogError(
                    "Token refresh attempt {RetryCount} failed after {Delay}s: {Message}",
                    retryCount, delay.TotalSeconds, exception.Message);
            }
        );
}
```

### 1.3 Non-Retryable Errors

```csharp
// Errors that should NOT be retried
private bool IsNonRetryable(HttpResponseMessage response)
{
    return response.StatusCode == HttpStatusCode.BadRequest ||          // 400
           response.StatusCode == HttpStatusCode.Unauthorized ||       // 401 (except for token refresh)
           response.StatusCode == HttpStatusCode.Forbidden ||           // 403
           response.StatusCode == HttpStatusCode.NotFound ||           // 404
           response.StatusCode == HttpStatusCode.Conflict;             // 409 (idempotency)
}
```

---

## 2. Circuit Breaker Pattern

### 2.1 Circuit Breaker Configuration

```csharp
// Circuit breaker for T-PayGate API endpoints
private IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return Policy
        .Handle<HttpRequestException>()
        .OrResult<HttpResponseMessage>(r => 
            r.StatusCode >= HttpStatusCode.InternalServerError ||  // 5xx
            r.StatusCode == HttpStatusCode.ServiceUnavailable)      // 503
        .CircuitBreakerAsync(
            exceptionsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (exception, breakDelay) =>
            {
                _logger.LogError(
                    "Circuit breaker tripped: {Message}. Break for {BreakDelay}s",
                    exception.Message, breakDelay.TotalSeconds);
                
                // Publish circuit breaker event
                _eventBus.Publish(new TpgCircuitBreakerTrippedEvent(exception.Message));
            },
            onReset: () =>
            {
                _logger.LogInformation("Circuit breaker reset - service recovered");
                _eventBus.Publish(new TpgCircuitBreakerResetEvent());
            },
            onHalfOpen: () =>
            {
                _logger.LogInformation("Circuit breaker half-open - testing service");
            }
        );
}
```

### 2.2 Circuit Breaker States

```csharp
public enum CircuitState
{
    Closed,    // Normal operation, requests pass through
    Open,       // Circuit tripped, requests fail immediately
    HalfOpen    // Testing if service recovered (3 requests max)
}
```

### 2.3 Circuit Breaker Monitoring

```csharp
public class CircuitBreakerMetrics
{
    public string ServiceName { get; set; }
    public CircuitState State { get; set; }
    public int FailureCount { get; set; }
    public int SuccessCount { get; set; }
    public DateTimeOffset? LastStateChange { get; set; }
    public DateTimeOffset? LastFailure { get; set; }
}
```

---

## 3. Timeout Patterns

### 3.1 Timeout Configuration by Endpoint

```csharp
public class TpgTimeoutConfig
{
    // OAuth endpoints
    public static readonly TimeSpan OAuthToken = TimeSpan.FromSeconds(5);
    
    // Bank connection endpoints (may involve bank API calls)
    public static readonly TimeSpan Connect = TimeSpan.FromSeconds(15);
    public static readonly TimeSpan ConfirmOtp = TimeSpan.FromSeconds(15);
    public static readonly TimeSpan Disconnect = TimeSpan.FromSeconds(10);
    
    // Bill operations
    public static readonly TimeSpan CreateBill = TimeSpan.FromSeconds(10);
    public static readonly TimeSpan QueryBill = TimeSpan.FromSeconds(5);
    
    // List operations
    public static readonly TimeSpan ListConnections = TimeSpan.FromSeconds(3);
    public static readonly TimeSpan ListBanks = TimeSpan.FromSeconds(3);
}
```

### 3.2 Timeout Policy Implementation

```csharp
private IAsyncPolicy<HttpResponseMessage> GetTimeoutPolicy(TimeSpan timeout)
{
    return Policy
        .TimeoutAsync<HttpResponseMessage>(timeout)
        .Or<TimeoutException>()
        .FallbackAsync(new HttpResponseMessage(HttpStatusCode.RequestTimeout)
        {
            Content = new StringContent("{\"error\":\"Request timeout\"}")
        });
}
```

---

## 4. Bulkhead Pattern

### 4.1 Resource Isolation

```csharp
// Separate connection pools for different T-PayGate operations
public class TpgConnectionPools
{
    // OAuth operations (low volume)
    public readonly IConnectionPool OAuthPool = new ConnectionPool(maxConnections: 2);
    
    // Bank connection operations (medium volume)
    public readonly IConnectionPool ConnectionPool = new ConnectionPool(maxConnections: 5);
    
    // Bill operations (high volume)
    public readonly IConnectionPool BillPool = new ConnectionPool(maxConnections: 20);
    
    // Webhook processing (high volume, isolated)
    public readonly IConnectionPool WebhookPool = new ConnectionPool(maxConnections: 10);
}
```

### 4.2 Thread Pool Isolation

```csharp
// Separate thread pools for different operation types
public class TpgThreadPoolConfig
{
    // API calls (normal priority)
    public readonly TaskScheduler ApiScheduler = 
        new LimitedConcurrencyLevelTaskScheduler(maxDegreeOfParallelism: 10);
    
    // Background processing (lower priority)
    public readonly TaskScheduler BackgroundScheduler = 
        new LimitedConcurrencyLevelTaskScheduler(maxDegreeOfParallelism: 5);
    
    // Webhook processing (high priority, isolated)
    public readonly TaskScheduler WebhookScheduler = 
        new LimitedConcurrencyLevelTaskScheduler(maxDegreeOfParallelism: 15);
}
```

---

## 5. Fallback Strategies

### 5.1 Fallback Hierarchy

```csharp
public class TpgFallbackStrategy
{
    // Primary: T-PayGate API
    private readonly ITpgApiClient _primaryClient;
    
    // Fallback 1: Cached data
    private readonly ITpgCacheService _cacheService;
    
    // Fallback 2: Alternative bank connection
    private readonly ITpgBankService _bankService;
    
    // Fallback 3: Degraded mode
    private readonly ITpgDegradedModeService _degradedService;
    
    public async Task<TpgBankList> GetBanksAsync()
    {
        try
        {
            // Primary: Call T-PayGate API
            return await _primaryClient.GetBanksAsync();
        }
        catch (Exception ex) when (IsTransientError(ex))
        {
            _logger.LogWarning("Primary API failed, using cached data: {Message}", ex.Message);
            
            // Fallback 1: Use cached data
            var cached = await _cacheService.GetBanksAsync();
            if (cached != null)
            {
                _logger.LogInformation("Using cached bank list");
                return cached;
            }
            
            // Fallback 2: Use degraded mode
            _logger.LogWarning("Cache miss, entering degraded mode");
            return await _degradedService.GetStaticBankListAsync();
        }
    }
}
```

### 5.2 Degraded Mode Implementation

```csharp
public class TpgDegradedModeService
{
    // Static fallback data
    private static readonly List<TpgBank> MajorBanks = new()
    {
        new TpgBank { Code = "VCB", Name = "Vietcombank", IsAvailable = true },
        new TpgBank { Code = "TCB", Name = "Techcombank", IsAvailable = true },
        new TpgBank { Code = "MB", Name = "MB Bank", IsAvailable = true }
    };
    
    public Task<List<TpgBank>> GetStaticBankListAsync()
    {
        _logger.LogWarning("Using degraded mode - static bank list");
        return Task.FromResult(MajorBanks);
    }
    
    public bool IsDegradedMode { get; private set; }
    
    public void EnableDegradedMode()
    {
        IsDegradedMode = true;
        _logger.LogWarning("Degraded mode enabled - functionality limited");
    }
    
    public void DisableDegradedMode()
    {
        IsDegradedMode = false;
        _logger.LogInformation("Degraded mode disabled - full functionality restored");
    }
}
```

---

## 6. Idempotency Patterns

### 6.1 Client-Side Idempotency

```csharp
public class TpgIdempotencyService
{
    // Cache for recent API calls (24-hour TTL)
    private readonly IMemoryCache _idempotencyCache;
    
    public async Task<TpgBillResponse> CreateBillWithIdempotencyAsync(CreateBillCommand command)
    {
        var cacheKey = $"bill:{command.RefTransactionId}";
        
        // Check if request was recently made
        if (_idempotencyCache.TryGetValue<TpgBillResponse>(cacheKey, out var cachedResponse))
        {
            _logger.LogInformation("Returning cached bill for refTransactionId: {RefId}", 
                command.RefTransactionId);
            return cachedResponse;
        }
        
        // Make actual API call
        var response = await _tpaygateClient.CreateBillAsync(command);
        
        // Cache response for 24 hours
        _idempotencyCache.Set(cacheKey, response, TimeSpan.FromHours(24));
        
        return response;
    }
}
```

### 6.2 Server-Side Idempotency

```csharp
public class WebhookIdempotencyService
{
    public async Task<WebhookProcessingResult> ProcessWebhookIdempotentlyAsync(TpgWebhookPayload payload)
    {
        // Check if already processed
        var existingNotification = await _notificationRepository.FindByBillCodeAsync(payload.BillCode);
        
        if (existingNotification != null && existingNotification.IsProcessed)
        {
            _logger.LogInformation("Duplicate webhook detected for billCode: {BillCode}", payload.BillCode);
            
            return new WebhookProcessingResult
            {
                IsDuplicate = true,
                OriginalProcessedAt = existingNotification.ProcessedAt,
                ShouldAcknowledge = true // Still return HTTP 200
            };
        }
        
        // Process new webhook
        return await ProcessNewWebhookAsync(payload);
    }
}
```

---

## 7. Monitoring and Health Checks

### 7.1 Health Check Implementation

```csharp
public class TpgHealthCheck : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        var checks = new Dictionary<string, object>
        {
            ["oauth_token"] = await CheckOAuthTokenAsync(),
            ["api_accessibility"] = await CheckApiAccessibilityAsync(),
            ["active_connections"] = await CountActiveConnectionsAsync(),
            ["circuit_breaker_state"] = GetCircuitBreakerState()
        };
        
        var isHealthy = checks.All(c => c.Value.ToString() == "Healthy");
        
        return isHealthy 
            ? HealthCheckResult.Healthy("T-PayGate integration healthy", checks)
            : HealthCheckResult.Degraded("T-PayGate integration degraded", checks);
    }
    
    private async Task<string> CheckOAuthTokenAsync()
    {
        if (_oauthService.IsTokenExpired())
        {
            try
            {
                await _oauthService.RefreshTokenAsync();
                return "Healthy";
            }
            catch
            {
                return "Unhealthy - Token refresh failed";
            }
        }
        return "Healthy";
    }
}
```

### 7.2 Metrics Collection

```csharp
public class TpgMetrics
{
    private readonly IMeter _meter;
    
    public TpgMetrics(IMeterFactory meterFactory)
    {
        _meter = meterFactory.CreateMeter("T-PayGate");
        
        // API call metrics
        ApiCalls = _meter.CreateCounter<int>("tpg_api_calls", "call", "operation");
        ApiLatency = _meter.CreateHistogram<double>("tpg_api_latency", "ms");
        ApiErrors = _meter.CreateCounter<int>("tpg_api_errors", "error", "operation");
        
        // Token metrics
        TokenRefreshes = _meter.CreateCounter<int>("tpg_token_refreshes");
        TokenFailures = _meter.CreateCounter<int>("tpg_token_failures");
        
        // Webhook metrics
        WebhooksReceived = _meter.CreateCounter<int>("tpg_webhooks_received");
        WebhooksProcessed = _meter.CreateCounter<int>("tpg_webhooks_processed");
        WebhooksDuplicate = _meter.CreateCounter<int>("tpg_webhooks_duplicate");
        WebhooksFailed = _meter.CreateCounter<int>("tpg_webhooks_failed");
        
        // Circuit breaker metrics
        CircuitBreakerTrips = _meter.CreateCounter<int>("tpg_circuit_breaker_trips");
        CircuitBreakerResets = _meter.CreateCounter<int>("tpg_circuit_breaker_resets");
    }
    
    public Counter<int> ApiCalls { get; }
    public Histogram<double> ApiLatency { get; }
    public Counter<int> ApiErrors { get; }
    
    public Counter<int> TokenRefreshes { get; }
    public Counter<int> TokenFailures { get; }
    
    public Counter<int> WebhooksReceived { get; }
    public Counter<int> WebhooksProcessed { get; }
    public Counter<int> WebhooksDuplicate { get; }
    public Counter<int> WebhooksFailed { get; }
    
    public Counter<int> CircuitBreakerTrips { get; }
    public Counter<int> CircuitBreakerResets { get; }
}
```

---

## 8. Composite Policy Wrappers

### 8.1 Full Resilience Pipeline

```csharp
public class TpgResilientClient
{
    private readonly IAsyncPolicy<HttpResponseMessage> _resiliencePipeline;
    
    public TpgResilientClient()
    {
        // Build composite policy: Timeout → Retry → Circuit Breaker → Fallback
        _resiliencePipeline = Policy
            .WrapAsync(GetCircuitBreakerPolicy())         // Outer: Circuit breaker
            .WrapAsync(GetRetryPolicy())                 // Middle: Retry
            .WrapAsync(GetTimeoutPolicy(TimeSpan.FromSeconds(10))); // Inner: Timeout
    }
    
    public async Task<HttpResponseMessage> ExecuteAsync(Func<Task<HttpResponseMessage>> action)
    {
        return await _resiliencePipeline.ExecuteAsync(action);
    }
}
```

### 8.2 Webhook-Specific Resilience

```csharp
public class TpgWebhookResilientHandler
{
    private readonly IAsyncPolicy<WebhookProcessingResult> _webhookPipeline;
    
    public TpgWebhookResilientHandler()
    {
        // Webhook-specific resilience: Immediate response, background processing
        _webhookPipeline = Policy
            .Handle<Exception>()
            .FallbackAsync(new WebhookProcessingResult { ShouldAcknowledge = true }) // Always return 200
            .WrapAsync(Policy.TimeoutAsync<WebhookProcessingResult>(TimeSpan.FromSeconds(8))); // 8s timeout (2s margin)
    }
    
    public async Task<HttpResponseMessage> HandleWebhookAsync(TpgWebhookPayload payload)
    {
        // Immediate response
        var response = new HttpResponseMessage(HttpStatusCode.OK);
        
        // Background processing with resilience
        _ = Task.Run(async () =>
        {
            await _webhookPipeline.ExecuteAsync(async () =>
            {
                return await ProcessWebhookBackgroundAsync(payload);
            });
        });
        
        return response;
    }
}
```

---

## 9. Chaos Engineering

### 9.1 Fault Injection Testing

```csharp
public class TpgChaosEngine
{
    private readonly Random _random = new();
    
    public async Task<T> ExecuteWithChaosAsync<T>(Func<Task<T>> operation, 
        double chaosProbability = 0.1)
    {
        // Randomly inject faults for testing resilience
        if (_random.NextDouble() < chaosProbability)
        {
            var faultType = _random.Next(3);
            
            switch (faultType)
            {
                case 0:
                    _logger.LogWarning("Chaos: Injecting delay");
                    await Task.Delay(TimeSpan.FromSeconds(5));
                    break;
                    
                case 1:
                    _logger.LogWarning("Chaos: Injecting exception");
                    throw new HttpRequestException("Chaos exception");
                    
                case 2:
                    _logger.LogWarning("Chaos: Injecting timeout");
                    await Task.Delay(TimeSpan.FromSeconds(30));
                    break;
            }
        }
        
        return await operation();
    }
}
```

---

## Related Documents
- [T-PayGate API Documentation](../../api/external/tpaygate.md) - API reference
- [T-PayGate Domain Overview](../../domains/payment/tpaygate/overview.md) - Domain context
- [T-PayGate Business Rules](../../domains/payment/tpaygate/business-rules.md) - Business rules for resiliency
- [T-PayGate Security](./tpaygate-security.md) - Security considerations
