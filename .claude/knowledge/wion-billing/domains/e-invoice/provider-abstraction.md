# Electronic Invoice Provider Abstraction

## Overview
This document describes the reusable Electronic Invoice SDK architecture that abstracts multiple electronic invoice providers (WiInvoice, VNPT, Viettel, MISA, etc.) behind a unified interface.

## Architecture Goals

1. **Provider Independence**: Easy to add new providers without changing application code
2. **Configuration Flexibility**: Support multiple providers with different configurations
3. **Error Resilience**: Handle provider-specific failures gracefully
4. **Testability**: Mock provider interfaces for testing
5. **Security**: Secure credential management per tenant

## SDK Location
```
src/framework/Wion.BuildingBlock/External/Invoice/
```

## Core Interfaces

### IElectronicInvoiceProvider
```csharp
namespace Wion.BuildingBlock.External.Invoice;

public interface IElectronicInvoiceProvider
{
    /// <summary>
    /// Creates a new electronic invoice with the provider
    /// </summary>
    Task<InvoiceCreationResult> CreateInvoiceAsync(
        InvoiceCreationRequest request,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets the current status of an invoice from the provider
    /// </summary>
    Task<InvoiceStatusResult> GetInvoiceStatusAsync(
        string providerInvoiceId,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Cancels an existing invoice
    /// </summary>
    Task<InvoiceCancellationResult> CancelInvoiceAsync(
        string providerInvoiceId,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets the PDF representation of an invoice
    /// </summary>
    Task<InvoicePdfResult> GetInvoicePdfAsync(
        string providerInvoiceId,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Validates an incoming webhook signature from the provider
    /// </summary>
    Task<bool> ValidateWebhookSignatureAsync(
        HttpContext context,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets available invoice templates from the provider
    /// </summary>
    Task<InvoiceTemplatesResult> GetTemplatesAsync(
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Tests the connection to the provider
    /// </summary>
    Task<ProviderHealthResult> TestConnectionAsync(
        CancellationToken cancellationToken = default);
}
```

### IElectronicInvoiceClientFactory
```csharp
public interface IElectronicInvoiceClientFactory
{
    /// <summary>
    /// Creates a provider client for the specified provider type
    /// </summary>
    IElectronicInvoiceProvider CreateClient(InvoiceProvider providerType);

    /// <summary>
    /// Creates a provider client with custom configuration
    /// </summary>
    IElectronicInvoiceProvider CreateClient(
        InvoiceProvider providerType,
        ProviderConfiguration configuration);
}
```

### IElectronicInvoiceConfigurationStore
```csharp
public interface IElectronicInvoiceConfigurationStore
{
    /// <summary>
    /// Gets provider configuration for a tenant
    /// </summary>
    Task<ProviderConfiguration> GetConfigurationAsync(
        string tenantId,
        InvoiceProvider provider,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Saves provider configuration for a tenant
    /// </summary>
    Task SaveConfigurationAsync(
        string tenantId,
        ProviderConfiguration configuration,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets the active provider for a tenant
    /// </summary>
    Task<InvoiceProvider> GetActiveProviderAsync(
        string tenantId,
        CancellationToken cancellationToken = default);
}
```

## Request/Response Models

### InvoiceCreationRequest
```csharp
public class InvoiceCreationRequest
{
    public string InvoiceNumber { get; set; }
    public DateTime IssuedDate { get; set; }
    public string CurrencyCode { get; set; } = "VND";
    public string TemplateCode { get; set; }
    public CustomerInfo Customer { get; set; }
    public SellerInfo Seller { get; set; }
    public List<InvoiceLineItem> LineItems { get; set; }
    public InvoicePaymentInfo Payment { get; set; }
    public Dictionary<string, string> Metadata { get; set; }
}

public class CustomerInfo
{
    public CustomerType Type { get; set; }
    public string DisplayName { get; set; }
    public string LegalName { get; set; }
    public string TaxCode { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
    public string Address { get; set; }
    public string IdentityCard { get; set; }
}

public class SellerInfo
{
    public string Name { get; set; }
    public string TaxCode { get; set; }
    public string Address { get; set; }
    public string Phone { get; set; }
}

public class InvoiceLineItem
{
    public string Code { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal DiscountPercentage { get; set; }
    public decimal DiscountAmount { get; set; }
    public int TaxRate { get; set; }
    public string Unit { get; set; }
}

public class InvoicePaymentInfo
{
    public decimal TotalAmountWithoutVat { get; set; }
    public decimal VatAmount { get; set; }
    public decimal TotalAmount { get; set; }
    public decimal DiscountAmount { get; set; }
}
```

### InvoiceCreationResult
```csharp
public class InvoiceCreationResult
{
    public bool IsSuccess { get; set; }
    public string ProviderInvoiceId { get; set; }
    public string InvoiceNumber { get; set; }
    public string TrackingUrl { get; set; }
    public InvoiceStatus Status { get; set; }
    public string ErrorMessage { get; set; }
    public Dictionary<string, object> ProviderData { get; set; }
}
```

### InvoiceStatusResult
```csharp
public class InvoiceStatusResult
{
    public bool IsSuccess { get; set; }
    public string ProviderInvoiceId { get; set; }
    public InvoiceStatus Status { get; set; }
    public string StatusName { get; set; }
    public DateTime? ApprovedDate { get; set; }
    public string ErrorMessage { get; set; }
}
```

## Configuration Models

### ProviderConfiguration
```csharp
public class ProviderConfiguration
{
    public InvoiceProvider ProviderType { get; set; }
    public string ApiKey { get; set; }
    public string ApiSecret { get; set; }
    public string PublicDomain { get; set; }
    public string TaxCode { get; set; }
    public Dictionary<string, string> Endpoints { get; set; }
    public Dictionary<string, string> AdditionalSettings { get; set; }
}
```

### ElectronicInvoiceOptions
```csharp
public class ElectronicInvoiceOptions
{
    public InvoiceProvider DefaultProvider { get; set; }
    public int MaxRetryAttempts { get; set; } = 3;
    public TimeSpan InitialRetryDelay { get; set; } = TimeSpan.FromSeconds(1);
    public TimeSpan MaxRetryDelay { get; set; } = TimeSpan.FromMinutes(5);
    public bool AutoIssueEnabled { get; set; } = true;
    public TimeSpan WebhookTimeout { get; set; } = TimeSpan.FromSeconds(30);
}
```

## Provider Implementations

### WiInvoiceProvider
```csharp
public class WiInvoiceProvider : IElectronicInvoiceProvider
{
    private readonly HttpClient _httpClient;
    private readonly ProviderConfiguration _configuration;
    private readonly ILogger<WiInvoiceProvider> _logger;

    public WiInvoiceProvider(
        HttpClient httpClient,
        IOptions<ProviderConfiguration> configuration,
        ILogger<WiInvoiceProvider> logger)
    {
        _httpClient = httpClient;
        _configuration = configuration.Value;
        _logger = logger;
    }

    public async Task<InvoiceCreationResult> CreateInvoiceAsync(
        InvoiceCreationRequest request,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var wiRequest = MapToWiInvoiceRequest(request);
            var signature = GenerateSignature(wiRequest);

            _httpClient.DefaultRequestHeaders.Clear();
            _httpClient.DefaultRequestHeaders.Add("x-api-key", _configuration.ApiKey);
            _httpClient.DefaultRequestHeaders.Add("x-signature", signature);
            _httpClient.DefaultRequestHeaders.Add("x-tenant", GetTenantId());

            var response = await _httpClient.PostAsJsonAsync(
                _configuration.Endpoints["CreateInvoice"],
                wiRequest,
                cancellationToken);

            var wiResponse = await response.Content.ReadFromJsonAsync<WiInvoiceResponse<WiInvoiceData>>(cancellationToken);

            return MapToInvoiceCreationResult(wiResponse);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "HTTP error while creating invoice");
            return new InvoiceCreationResult
            {
                IsSuccess = false,
                ErrorMessage = "Provider communication failed"
            };
        }
    }

    public async Task<bool> ValidateWebhookSignatureAsync(
        HttpContext context,
        CancellationToken cancellationToken = default)
    {
        // WiInvoice signature validation logic
        return true;
    }

    private string GenerateSignature(WiInvoiceRequest request)
    {
        var message = $"{AppCode}_{ApiKey}_{TenantId}_{JsonConvert.SerializeObject(request)}";
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(_configuration.ApiSecret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(message));
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }

    // Other implementation methods...
}
```

## Retry Strategy

### Exponential Backoff Retry Handler
```csharp
public class ElectronicInvoiceRetryHandler
{
    private readonly ElectronicInvoiceOptions _options;
    private readonly ILogger<ElectronicInvoiceRetryHandler> _logger;

    public async Task<T> ExecuteWithRetryAsync<T>(
        Func<Task<T>> operation,
        string operationName,
        CancellationToken cancellationToken = default)
    {
        var attempt = 0;
        var delay = _options.InitialRetryDelay;

        while (attempt < _options.MaxRetryAttempts)
        {
            try
            {
                return await operation();
            }
            catch (HttpRequestException ex) when (IsTransientError(ex))
            {
                attempt++;
                _logger.LogWarning(
                    "Transient error during {OperationName} (attempt {Attempt}/{MaxAttempts}): {Error}",
                    operationName, attempt, _options.MaxRetryAttempts, ex.Message);

                if (attempt >= _options.MaxRetryAttempts)
                    throw;

                await Task.Delay(delay, cancellationToken);
                delay = TimeSpan.FromSeconds(
                    Math.Min(delay.TotalSeconds * 2, _options.MaxRetryDelay.TotalSeconds));
            }
        }

        throw new InvalidOperationException("Max retry attempts exceeded");
    }

    private bool IsTransientError(HttpRequestException ex)
    {
        return ex.StatusCode.HasValue &&
               (ex.StatusCode.Value == System.Net.HttpStatusCode.RequestTimeout ||
                ex.StatusCode.Value == System.Net.HttpStatusCode.ServiceUnavailable ||
                ex.StatusCode.Value >= 500);
    }
}
```

## Error Handling

### Provider Error Codes
```csharp
public static class ElectronicInvoiceErrorCodes
{
    // Provider communication errors
    public const string ProviderUnavailable = "Invoice.Provider:0001";
    public const string ProviderTimeout = "Invoice.Provider:0002";
    public const string ProviderAuthenticationFailed = "Invoice.Provider:0003";

    // Invoice validation errors
    public const string InvalidInvoiceData = "Invoice.Validation:0001";
    public const string MissingCustomerInfo = "Invoice.Validation:0002";
    public const string InvalidTaxCode = "Invoice.Validation:0003";

    // Configuration errors
    public const string ProviderNotConfigured = "Invoice.Config:0001";
    public const string InvalidProviderConfiguration = "Invoice.Config:0002";

    // Operation errors
    public const string InvoiceAlreadyExists = "Invoice.Operation:0001";
    public const string InvoiceCannotBeCancelled = "Invoice.Operation:0002";
    public const string InvoiceGenerationFailed = "Invoice.Operation:0003";
}
```

## Dependency Injection Setup

```csharp
// In Wion.BuildingBlock/External/Invoice/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddElectronicInvoiceProvider(
        this IServiceCollection services,
        InvoiceProvider providerType,
        Action<ProviderConfiguration> configure = null)
    {
        services.Configure<ElectronicInvoiceOptions>(options =>
        {
            // Default options
        });

        switch (providerType)
        {
            case InvoiceProvider.WiInvoice:
                services.AddHttpClient<WiInvoiceProvider>();
                services.AddSingleton<IElectronicInvoiceProvider, WiInvoiceProvider>();
                break;

            case InvoiceProvider.VNPT:
                services.AddHttpClient<VNPTInvoiceProvider>();
                services.AddSingleton<IElectronicInvoiceProvider, VNPTInvoiceProvider>();
                break;

            // Add other providers...
        }

        return services;
    }
}
```

## Usage Example

```csharp
// In application service
public class ElectronicInvoiceAppService
{
    private readonly IElectronicInvoiceClientFactory _providerFactory;
    private readonly IElectronicInvoiceConfigurationStore _configStore;

    public async Task<string> GenerateInvoiceAsync(CreateInvoiceDto input)
    {
        // Get provider configuration for tenant
        var providerType = await _configStore.GetActiveProviderAsync(CurrentTenant.Id);
        var configuration = await _configStore.GetConfigurationAsync(CurrentTenant.Id, providerType);

        // Create provider client
        var provider = _providerFactory.CreateClient(providerType, configuration);

        // Create invoice
        var request = new InvoiceCreationRequest
        {
            InvoiceNumber = input.InvoiceNumber,
            Customer = input.Customer,
            // ... other properties
        };

        var result = await provider.CreateInvoiceAsync(request);

        if (!result.IsSuccess)
        {
            throw new BusinessException(result.ErrorMessage);
        }

        return result.ProviderInvoiceId;
    }
}
```

## Testing Strategy

### Mock Provider for Testing
```csharp
public class MockElectronicInvoiceProvider : IElectronicInvoiceProvider
{
    public Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request, CancellationToken cancellationToken = default)
    {
        return Task.FromResult(new InvoiceCreationResult
        {
            IsSuccess = true,
            ProviderInvoiceId = "MOCK-" + Guid.NewGuid(),
            InvoiceNumber = request.InvoiceNumber,
            Status = InvoiceStatus.Approved
        });
    }

    // Implement other methods...
}
```

---

## Next Steps
- [WiInvoice Provider Implementation](./providers/wiinvoice.md)
- [VNPT Provider Implementation](./providers/vnpt.md)
- [Integration Testing](./testing.md)
- [Security Considerations](./security.md)
