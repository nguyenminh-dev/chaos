# E-Invoice Integration

## Overview
This document describes the integration patterns for e-invoice providers based on analysis of the billing-management service implementation.

## Provider Integration Architecture

### Integration Pattern Overview
```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   Application   │───────▶  │ Invoice Provider │────────▶│  Invoice Hub    │
│   (Invoice      │  HTTP   │  (WiInvoice,     │  HTTPS  │  (Tax Authority)│
│    Reference)   │  API    │   VNPT, etc.)    │         │                 │
└─────────────────┘         └──────────────────┘         └─────────────────┘
        │                             │
        │                             │
        ▼                             ▼
┌─────────────────┐         ┌──────────────────┐
│   Database      │◀────────│   Webhook        │
│   (Invoice      │ Callback │   (Status        │
│    Reference)   │         │    Updates)      │
└─────────────────┘         └──────────────────┘
```

### Integration Patterns from billing-management

#### 1. Provider Client Pattern
**Purpose**: Abstract external provider APIs behind service interfaces

**Implementation**:
```csharp
// Provider Service Interface
public interface IEInvoiceProvider
{
    Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request);
    Task<InvoiceStatusResult> GetInvoiceStatusAsync(string providerInvoiceId);
    Task<InvoiceCancellationResult> CancelInvoiceAsync(string providerInvoiceId);
    Task<InvoicePdfResult> GetInvoicePdfAsync(string providerInvoiceId);
}

// WiInvoice Implementation
public class WiInvoiceService : IElectronicInvoiceProvider
{
    private readonly HttpClient _httpClient;
    private readonly WiInvoiceOption _options;
    
    public async Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request)
    {
        var signature = GenerateSignature(request);
        
        _httpClient.DefaultRequestHeaders.Add("x-api-key", _options.ApiKey);
        _httpClient.DefaultRequestHeaders.Add("x-signature", signature);
        _httpClient.DefaultRequestHeaders.Add("x-tenant", CurrentTenant.Id);
        
        var response = await _httpClient.PostAsJsonAsync(_options.CreateInvoice, request);
        var result = await response.Content.ReadFromJsonAsync<WiInvoiceResponse<InvoiceCreationResult>>();
        
        return MapToInvoiceCreationResult(result);
    }
}
```

#### 2. Authentication Pattern
**Purpose**: Secure API communication with electronic invoice providers

**HMAC-SHA256 Signature**:
```csharp
private string GenerateSignature(HttpMethod method, string tenant, string body = null)
{
    var message = method == HttpMethod.Post 
        ? $"{AppCode}_{ApiKey}_{tenant}_{body}"
        : $"{AppCode}_{ApiKey}_{tenant}";
    
    using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(ApiSecret));
    var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(message));
    return hash.Aggregate("", (s, e) => s + $"{e:x2}", s => s);
}
```

**Request Headers**:
```
x-api-key: {provider_api_key}
x-signature: {hmac_sha256_signature}
x-tenant: {tenant_id}
x-config-id: {provider_config_id}
Accept-Language: {culture_code}
```

#### 3. Configuration Pattern
**Purpose**: Manage provider settings per tenant

**Configuration Model**:
```csharp
public class WiInvoiceOption
{
    public string PublicDomain { get; set; }
    public string AppCode { get; set; }
    public string ApiKey { get; set; }
    public string ApiSecret { get; set; }
    public string UrlWebhook { get; set; }
    public string Connect { get; set; }
    public string Setting { get; set; }
    public string Disconnect { get; set; }
    public string CreateInvoice { get; set; }
    public string GetListInvoice { get; set; }
    public string GetInvoice { get; set; }
    public string UpdateInvoice { get; set; }
    public string DeleteInvoice { get; set; }
    public string PrintPdf { get; set; }
}
```

**appsettings.json**:
```json
{
  "Services": {
    "WiInvoiceClient": {
      "PublicDomain": "https://api.wiinvoice.vn",
      "AppCode": "WION_BILLING",
      "ApiKey": "{api_key}",
      "ApiSecret": "{api_secret}",
      "UrlWebhook": "https://billing.wion.vn/webhook/invoice",
      "Connect": "/api/v1/connect",
      "CreateInvoice": "/api/v1/invoices/create",
      "GetInvoice": "/api/v1/invoices/get",
      "UpdateInvoice": "/api/v1/invoices/update",
      "DeleteInvoice": "/api/v1/invoices/delete",
      "PrintPdf": "/api/v1/invoices/pdf"
    }
  }
}
```

#### 4. HttpClient Factory Pattern
**Purpose**: Create and configure HTTP clients for provider communication

**Implementation**:
```csharp
// In Startup/Module configuration
private void ConfigureHttpClient(ServiceConfigurationContext context, IConfiguration configuration)
{
    var httpClientBuilder = context.Services.AddHttpClient("WiInvoiceClient", client =>
    {
        client.BaseAddress = new Uri(configuration["Services:WiInvoiceClient:PublicDomain"]);
        client.Timeout = TimeSpan.FromSeconds(30);
    });
    
    // Configure retry policy
    httpClientBuilder.AddTransientHttpErrorPolicy(policy =>
        policy.WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));
}

// In service usage
private readonly IHttpClientFactory _httpClientFactory;

public WiInvoiceService(IHttpClientFactory httpClientFactory)
{
    _httpClient = _httpClientFactory.CreateClient("WiInvoiceClient");
}
```

#### 5. Request/Response Pattern
**Purpose**: Standardized data exchange with providers

**Create Invoice Request**:
```csharp
public class InvoiceCreationRequest
{
    public bool IsSign { get; set; } // Submit to tax authority
    public DateTime IssuedDate { get; set; }
    public string Template { get; set; }
    public string CurrencyCode { get; set; }
    public decimal ExchangeRate { get; set; }
    public string OrderNumber { get; set; }
    
    public CustomerInfo Buyer { get; set; }
    public PaymentInfo Payment { get; set; }
    public SellerInfo Seller { get; set; }
    public DeliveryInfo Delivery { get; set; }
    public List<LineItem> Details { get; set; }
}
```

**Provider Response Wrapper**:
```csharp
public class WiInvoiceResponse<T> where T : class
{
    public T Data { get; set; }
    public bool Succeeded { get; set; }
    public string Error { get; set; }
}
```

#### 6. Error Handling Pattern
**Purpose**: Graceful error handling with retry logic

**Implementation**:
```csharp
public async Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request)
{
    try
    {
        var response = await _wiInvoiceClient.SendAsync(request);
        
        if (!response.IsSuccessStatusCode || !response.Succeeded)
        {
            throw new BusinessException(
                BillingDomainErrorCodes.InvoiceCreationFailed,
                $"Provider error: {response.Error}");
        }
        
        return response.Data;
    }
    catch (HttpRequestException ex)
    {
        throw new BusinessException(
            BillingDomainErrorCodes.ProviderCommunicationFailed,
            $"Failed to communicate with provider: {ex.Message}");
    }
    catch (TaskCanceledException ex)
    {
        throw new BusinessException(
            BillingDomainErrorCodes.ProviderTimeout,
            $"Provider request timed out: {ex.Message}");
    }
}
```

#### 7. Webhook Pattern
**Purpose**: Receive asynchronous status updates from providers

**Implementation**:
```csharp
[Route("webhook/invoice")]
[IgnoreAntiforgeryToken]
public class WebhookController : AbpController
{
    private readonly IInvoiceAppService _invoiceAppService;
    private readonly ILogger<WebhookController> _logger;

    [HttpPost]
    public async Task<IActionResult> InvoiceUpdateStatusAsync([FromBody] InvoiceWebhookDto input)
    {
        try
        {
            // Validate webhook signature
            if (!await ValidateWebhookSignatureAsync(Request))
            {
                return Unauthorized(new { success = false, message = "Invalid signature" });
            }

            await _invoiceAppService.UpdateInvoiceStatusAsync(input);
            
            _logger.LogInformation(
                "Invoice webhook processed: OrderNumber={OrderNumber}, Key={Key}, Status={Status}",
                input.OrderNumber, input.Key, input.StatusName);
                
            return Ok(new { success = true });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Webhook processing failed");
            return StatusCode(500, new { success = false, message = ex.Message });
        }
    }
}
```

**Webhook DTO**:
```csharp
public class InvoiceWebhookDto
{
    public string OrderNumber { get; set; }
    public string Key { get; set; } // Provider invoice ID
    public string StatusName { get; set; }
    public InvoiceStatus StatusCode { get; set; }
}
```

#### 8. Idempotency Pattern
**Purpose**: Prevent duplicate invoice generation

**Implementation**:
```csharp
public async Task<string> GenerateInvoiceAsync(CreateInvoiceDto input)
{
    // Check for existing invoice
    var existingInvoice = await _invoiceRepository.FindByOrderNumberAsync(input.OrderNumber);
    if (existingInvoice != null)
    {
        throw new BusinessException(
            BillingDomainErrorCodes.InvoiceAlreadyExists,
            $"Invoice for order {input.OrderNumber} already exists");
    }

    // Generate invoice with provider
    var result = await _provider.CreateInvoiceAsync(input);
    
    // Store invoice reference
    var invoice = InvoiceReference.Create(
        input.OrderNumber,
        result.ProviderInvoiceId,
        result.TrackingUrl);
    
    await _invoiceRepository.AddAsync(invoice);
    
    return result.ProviderInvoiceId;
}
```

#### 9. Background Processing Pattern
**Purpose**: Asynchronous invoice generation and status sync

**Event-Driven Approach**:
```csharp
// Domain event
public class PaymentCompletedDomainEvent : IDomainEvent
{
    public string PaymentId { get; set; }
    public decimal Amount { get; set; }
    public string OrderNumber { get; set; }
    public CustomerInfo Customer { get; set; }
}

// Event handler
public class PaymentCompletedHandler : IDistributedEventHandler<PaymentCompletedEvent>
{
    private readonly IInvoiceAppService _invoiceAppService;

    public async Task HandleEventAsync(PaymentCompletedEvent eventData)
    {
        if (eventData.AutoGenerateInvoice)
        {
            await _invoiceAppService.GenerateInvoiceAsync(new GenerateInvoiceCommand
            {
                OrderNumber = eventData.OrderNumber,
                Amount = eventData.Amount,
                Customer = eventData.Customer
            });
        }
    }
}
```

#### 10. Retry Strategy Pattern
**NOTE**: Retry logic has been removed from the invoice creation process. Failed invoices are marked as `Failed` status immediately without automatic retry attempts.

**Updated Error Handling**:
```csharp
public class InvoiceErrorHandler
{
    private readonly ILogger<InvoiceErrorHandler> _logger;
    
    public async Task HandleInvoiceCreationErrorAsync(
        Func<Task<InvoiceCreationResult>> operation,
        string operationName,
        CancellationToken cancellationToken = default)
    {
        try
        {
            return await operation();
        }
        catch (HttpRequestException ex) when (IsTransientError(ex))
        {
            _logger.LogError(
                "Transient error during {Operation}: {Error}",
                operationName, ex.Message);

            throw new BusinessException(
                BillingDomainErrorCodes.InvoiceGenerationFailed,
                $"Provider communication failed: {ex.Message}");
        }
        catch (Exception ex)
        {
            _logger.LogError(
                "Failed to create invoice during {Operation}: {Error}",
                operationName, ex.Message);

            throw new BusinessException(
                BillingDomainErrorCodes.InvoiceGenerationFailed,
                $"Invoice generation failed: {ex.Message}");
        }
    }
}
```

## Provider Operations

### 1. Connect to Provider
**Purpose**: Establish connection and obtain configuration ID

```csharp
public async Task<ProviderConnectionResult> ConnectAsync(ConnectProviderDto input)
{
    var request = new ProviderConnectRequest
    {
        TenantName = CurrentTenant.Id,
        UrlWebhook = _options.UrlWebhook,
        TaxCode = input.TaxCode,
        Username = input.Username,
        Password = input.Password
    };

    var result = await _provider.ConnectAsync(request);
    
    // Store provider configuration
    var settings = new InvoiceProviderSettings
    {
        TenantId = CurrentTenant.Id,
        Provider = InvoiceProvider.WiInvoice,
        ConfigId = result.ConfigId,
        TaxCode = result.TaxCode,
        Username = input.Username
    };

    await _settingsRepository.AddAsync(settings);
    
    return result;
}
```

### 2. Create Invoice
**Purpose**: Generate e-invoice with provider

```csharp
public async Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request)
{
    // Validate invoice data
    ValidateInvoiceRequest(request);

    // Get provider settings
    var settings = await _settingsRepository.FindActiveAsync(CurrentTenant.Id);
    if (settings == null)
    {
        throw new BusinessException(BillingDomainErrorCodes.ProviderNotConfigured);
    }

    // Transform request to provider format
    var providerRequest = MapToProviderRequest(request, settings);

    // Execute without retry logic
    try
    {
        var result = await _provider.CreateInvoiceAsync(providerRequest);

        // Store invoice reference
        var invoice = InvoiceReference.Create(
            request.OrderNumber,
            result.ProviderInvoiceId,
            result.TrackingUrl);

        await _invoiceRepository.AddAsync(invoice);

        return result;
    }
    catch (Exception ex)
    {
        // Mark as failed - no retry logic
        var invoice = InvoiceReference.Create(request.OrderNumber);
        invoice.Fail(ex.Message);
        await _invoiceRepository.AddAsync(invoice);

        throw;
    }
}
```

### 3. Get Invoice Status
**Purpose**: Synchronize invoice status from provider

```csharp
public async Task<InvoiceStatusResult> GetInvoiceStatusAsync(string providerInvoiceId)
{
    var result = await _provider.GetInvoiceStatusAsync(providerInvoiceId);

    // Update local invoice reference
    var invoice = await _invoiceRepository.FindByProviderInvoiceIdAsync(providerInvoiceId);
    if (invoice != null)
    {
        invoice.UpdateStatus(result.Status, result.StatusName);
        await _invoiceRepository.UpdateAsync(invoice);
    }

    return result;
}
```

### 4. Update Invoice
**Purpose**: Modify pending invoice details

```csharp
public async Task<InvoiceUpdateResult> UpdateInvoiceAsync(string providerInvoiceId, InvoiceUpdateRequest request)
{
    // Validate invoice can be updated
    var invoice = await _invoiceRepository.FindByProviderInvoiceIdAsync(providerInvoiceId);
    if (invoice == null)
    {
        throw new BusinessException(BillingDomainErrorCodes.InvoiceNotFound);
    }

    if (invoice.Status != InvoiceStatus.WaitSign)
    {
        throw new BusinessException(BillingDomainErrorCodes.InvalidInvoiceStatusForUpdate);
    }

    var result = await _provider.UpdateInvoiceAsync(providerInvoiceId, request);

    // Update local invoice
    invoice.UpdateDetails(request);
    await _invoiceRepository.UpdateAsync(invoice);

    return result;
}
```

### 5. Cancel Invoice
**Purpose**: Cancel issued invoice

```csharp
public async Task CancelInvoiceAsync(string providerInvoiceId)
{
    // Validate invoice can be cancelled
    var invoice = await _invoiceRepository.FindByProviderInvoiceIdAsync(providerInvoiceId);
    if (invoice == null)
    {
        throw new BusinessException(BillingDomainErrorCodes.InvoiceNotFound);
    }

    if (invoice.Status != InvoiceStatus.WaitSign)
    {
        throw new BusinessException(BillingDomainErrorCodes.InvalidInvoiceStatusForCancel);
    }

    await _provider.CancelInvoiceAsync(providerInvoiceId);

    // Update local invoice
    invoice.Cancel();
    await _invoiceRepository.UpdateAsync(invoice);
}
```

### 6. Get Invoice PDF
**Purpose**: Download PDF representation of issued invoice

```csharp
public async Task<InvoicePdfResult> GetInvoicePdfAsync(string providerInvoiceId)
{
    // Validate invoice is issued
    var invoice = await _invoiceRepository.FindByProviderInvoiceIdAsync(providerInvoiceId);
    if (invoice == null || invoice.Status != InvoiceStatus.Issued)
    {
        throw new BusinessException(BillingDomainErrorCodes.InvalidInvoiceStatusForPdf);
    }

    var result = await _provider.GetInvoicePdfAsync(providerInvoiceId);
    
    return result;
}
```

## Data Synchronization

### Webhook-Based Status Sync
```csharp
public async Task HandleWebhookAsync(InvoiceWebhookDto webhookData)
{
    // Find invoice by provider invoice ID
    var invoice = await _invoiceRepository.FindByProviderInvoiceIdAsync(webhookData.Key);
    if (invoice == null)
    {
        _logger.LogWarning("Invoice not found for webhook: {Key}", webhookData.Key);
        return;
    }

    // Update invoice status
    var oldStatus = invoice.Status;
    invoice.UpdateStatus(webhookData.StatusCode, webhookData.StatusName);
    
    await _invoiceRepository.UpdateAsync(invoice);

    // Publish domain event
    if (oldStatus != invoice.Status)
    {
        await _eventBus.PublishAsync(new InvoiceStatusChangedDomainEvent
        {
            InvoiceId = invoice.Id,
            OldStatus = oldStatus,
            NewStatus = invoice.Status,
            ProviderInvoiceId = invoice.ProviderInvoiceId
        });
    }
}
```

### Polling-Based Status Sync (Fallback)
```csharp
public class InvoiceStatusSyncJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var pendingInvoices = await _invoiceRepository
                .FindPendingInvoicesAsync(DateTime.UtcNow.AddHours(-1));

            foreach (var invoice in pendingInvoices)
            {
                try
                {
                    var status = await _provider.GetInvoiceStatusAsync(invoice.ProviderInvoiceId);
                    invoice.UpdateStatus(status.Status, status.StatusName);
                    await _invoiceRepository.UpdateAsync(invoice);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to sync invoice status: {InvoiceId}", invoice.Id);
                }
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

## Security Considerations

### 1. Credential Management
```csharp
// Store encrypted credentials
public class InvoiceProviderSettings
{
    public string ApiKey { get; set; } // Encrypted at rest
    public string ApiSecret { get; set; } // Encrypted at rest
}

// Use Azure Key Vault or similar for production
public class CredentialManager
{
    public async Task<string> GetSecretAsync(string secretName)
    {
        return await _keyVaultClient.GetSecretAsync(secretName);
    }
}
```

### 2. Webhook Signature Validation
```csharp
public async Task<bool> ValidateWebhookSignatureAsync(HttpRequest request)
{
    var signature = request.Headers["X-Signature"].FirstOrDefault();
    var payload = await new StreamReader(request.Body).ReadToEndAsync();

    var expectedSignature = ComputeHmac(payload, _secret);
    
    return signature == expectedSignature;
}
```

### 3. Request Throttling
```csharp
public class ThrottlingDelegatingHandler : DelegatingHandler
{
    private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(10); // Max 10 concurrent requests

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        await _semaphore.WaitAsync(cancellationToken);
        try
        {
            return await base.SendAsync(request, cancellationToken);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## Monitoring and Logging

### 1. Provider Call Metrics
```csharp
public class MetricInterceptor : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            var response = await base.SendAsync(request, cancellationToken);
            
            _meterProvider.Record(
                "invoice.provider.calls",
                new Tag("provider", "wiinvoice"),
                new Tag("operation", request.RequestUri.AbsolutePath),
                new Tag("status", response.StatusCode.ToString()),
                new Tag("duration_ms", stopwatch.ElapsedMilliseconds));

            return response;
        }
        catch (Exception ex)
        {
            _meterProvider.Record(
                "invoice.provider.errors",
                new Tag("provider", "wiinvoice"),
                new Tag("error_type", ex.GetType().Name));
            throw;
        }
    }
}
```

### 2. Structured Logging
```csharp
public class EInvoiceService
{
    private readonly ILogger<EInvoiceService> _logger;

    public async Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request)
    {
        _logger.LogInformation(
            "Creating invoice: OrderNumber={OrderNumber}, Amount={Amount}, Customer={Customer}",
            request.OrderNumber, request.TotalAmount, request.Customer.TaxCode);

        try
        {
            var result = await _provider.CreateInvoiceAsync(request);

            _logger.LogInformation(
                "Invoice created successfully: OrderNumber={OrderNumber}, ProviderInvoiceId={ProviderInvoiceId}",
                request.OrderNumber, result.ProviderInvoiceId);

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to create invoice: OrderNumber={OrderNumber}, Error={Error}",
                request.OrderNumber, ex.Message);
            throw;
        }
    }
}
```

## Next Steps
- [Provider Implementation Guide](./provider-implementation.md)
- [Testing Strategy](./testing.md)
- [Deployment Guide](./deployment.md)
