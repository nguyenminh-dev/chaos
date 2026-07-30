# Invoice Business Rules

## Core Business Rules

### BR-I-001: One Invoice Per Payment
**Rule**: Each payment can have only one invoice

**Enforcement**: 
- Unique constraint on paymentId
- Repository check before invoice creation
- Idempotency in provider API calls

**Implementation**:
```csharp
public class InvoiceMustBeUniqueRule : IBusinessRule
{
    private readonly string _paymentId;
    private readonly IInvoiceReferenceRepository _repository;

    public bool IsBroken() => 
        _repository.ExistsByPaymentId(_paymentId);

    public string Message => 
        $"Invoice for payment {_paymentId} already exists";
}
```

---

### BR-I-002: Invoice Data Completeness
**Rule**: Invoice data must be complete before provider submission

**Required Fields**:
- **Customer Information**: Name, address, tax code (for businesses)
- **Seller Information**: Name, tax code, address
- **Financial Data**: Amount, tax amount, total amount
- **Line Items**: Product/service details with quantities and prices
- **Invoice Details**: Invoice number, issue date, template

**Enforcement**: 
- Validation before provider API call
- Domain model validation rules
- Provider-specific validation checks

**Implementation**:
```csharp
public class InvoiceDataCompletenessRule : IBusinessRule
{
    private readonly InvoiceCreationRequest _request;

    public bool IsBroken()
    {
        return string.IsNullOrEmpty(_request.Customer?.TaxCode) &&
               string.IsNullOrEmpty(_request.Customer?.Email) &&
               string.IsNullOrEmpty(_request.Customer?.Phone);
    }

    public string Message => 
        "Customer must have at least one: tax code, email, or phone";
}
```

---

### BR-I-003: Retry on Transient Failures
**Rule**: Failed invoice creation must be retried for transient errors

**Retry Strategy**:
- **Max Attempts**: 3
- **Backoff**: Exponential (1s, 2s, 4s)
- **Transient Errors**: Network timeouts, 5xx server errors
- **Permanent Errors**: Validation failures, authentication errors

**Enforcement**: 
- Automatic retry with exponential backoff
- Background job for scheduled retries
- Circuit breaker for provider unavailability

**Implementation**:
```csharp
public class InvoiceRetryHandler
{
    public async Task<T> ExecuteWithRetryAsync<T>(Func<Task<T>> operation)
    {
        var attempt = 0;
        var delay = TimeSpan.FromSeconds(1);
        var maxAttempts = 3;

        while (attempt < maxAttempts)
        {
            try
            {
                return await operation();
            }
            catch (HttpRequestException ex) when (IsTransientError(ex))
            {
                attempt++;
                if (attempt >= maxAttempts) throw;
                await Task.Delay(delay);
                delay = TimeSpan.FromSeconds(delay.TotalSeconds * 2);
            }
        }
        throw new InvalidOperationException("Max retry exceeded");
    }
}
```

---

### BR-I-004: Manual Queue After Max Retries
**Rule**: After 3 failed attempts, invoice queued for manual processing

**Purpose**: Ensure invoices are eventually issued even with provider issues

**Enforcement**: 
- Background job monitors failed invoices
- Manual processing queue for operator intervention
- Notification system for failed invoices

**Implementation**:
```csharp
public class FailedInvoiceMonitoringJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var failedInvoices = await _repository
            .FindFailedInvoicesAsync(DateTime.UtcNow.AddHours(-24));

        foreach (var invoice in failedInvoices)
        {
            await _notificationService.NotifyAdminsAsync(invoice);
        }

        await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
    }
}
```

---

### BR-I-005: Invoice Only on Payment Success
**Rule**: Invoice created only when payment succeeds

**Related**: [Invoice After Payment Policy](../../policies/invoice-after-payment.md)

**Enforcement**: 
- Event-driven architecture
- Domain event: PaymentCompleted → GenerateInvoice
- No invoice generation for failed/cancelled payments

**Implementation**:
```csharp
public class PaymentCompletedHandler : IDistributedEventHandler<PaymentCompletedEvent>
{
    public async Task HandleEventAsync(PaymentCompletedEvent eventData)
    {
        if (eventData.Status == PaymentStatus.Completed)
        {
            await _invoiceAppService.GenerateInvoiceAsync(
                new GenerateInvoiceCommand { PaymentId = eventData.PaymentId });
        }
    }
}
```

## Electronic Invoice Specific Rules

### BR-I-E-001: Provider Configuration Required
**Rule**: Electronic invoice generation requires valid provider configuration

**Requirements**:
- Active provider settings for tenant
- Valid API credentials
- Provider-specific tax code
- Configured webhook URL

**Enforcement**:
```csharp
public class ProviderMustBeConfiguredRule : IBusinessRule
{
    private readonly string _tenantId;
    private readonly IInvoiceSettingsRepository _repository;

    public bool IsBroken()
    {
        var settings = _repository.FindActiveByTenant(_tenantId);
        return settings == null || !settings.IsActive;
    }

    public string Message => 
        "Electronic invoice provider not configured for tenant";
}
```

---

### BR-I-E-002: Valid Invoice Number Format
**Rule**: Invoice numbers must follow Vietnam electronic invoice regulations

**Format**: `1K{template_code}/{year}-{sequential_number}`

**Validation**:
- Template code must match provider configuration
- Sequential number must be unique per tenant/year
- Year must be current or previous year

**Implementation**:
```csharp
public class InvoiceNumberFormatRule : IBusinessRule
{
    private readonly string _invoiceNumber;
    private readonly string _templateCode;

    public bool IsBroken()
    {
        var pattern = $@"^1K{_templateCode}/\d{{4}}-\d+$";
        return !Regex.IsMatch(_invoiceNumber, pattern);
    }

    public string Message => 
        $"Invoice number must follow format: 1K{_templateCode}/YYYY-XXXXX";
}
```

---

### BR-I-E-003: Invoice Status Transition Validation
**Rule**: Invoice status must follow valid state machine transitions

**Valid Transitions**:
- `Pending` → `Processing` → `Approved` → `Issued`
- `Pending` → `Failed` (permanent error)
- `Approved` → `Cancelled` (within valid timeframe)
- `Issued` → `Replaced` (with new invoice)

**Invalid Transitions**:
- `Failed` → `Issued` (must retry from Pending)
- `Cancelled` → `Issued` (cancelled invoices cannot be re-issued)
- `Issued` → `Pending` (cannot revert status)

**Implementation**:
```csharp
public class InvoiceStatusTransitionRule : IBusinessRule
{
    private readonly InvoiceReference _invoice;
    private readonly InvoiceStatus _newStatus;

    private static readonly Dictionary<InvoiceStatus, InvoiceStatus[]> ValidTransitions = new()
    {
        [InvoiceStatus.Pending] = new[] { InvoiceStatus.Processing, InvoiceStatus.Failed },
        [InvoiceStatus.Processing] = new[] { InvoiceStatus.Approved, InvoiceStatus.Failed },
        [InvoiceStatus.Approved] = new[] { InvoiceStatus.Issued, InvoiceStatus.Cancelled },
        [InvoiceStatus.Issued] = new[] { InvoiceStatus.Replaced },
        [InvoiceStatus.Failed] = new[] { InvoiceStatus.Pending }, // Retry allowed
    };

    public bool IsBroken()
    {
        if (!ValidTransitions.ContainsKey(_invoice.Status))
            return true;

        return !ValidTransitions[_invoice.Status].Contains(_newStatus);
    }

    public string Message => 
        $"Invalid status transition: {_invoice.Status} → {_newStatus}";
}
```

---

### BR-I-E-004: Invoice Cancellation Constraints
**Rule**: Only invoices in specific statuses can be cancelled

**Cancellable Statuses**:
- `Approved` (before final issuance)
- `Issued` (within 24 hours of issuance, per tax regulations)

**Non-Cancellable Statuses**:
- `Pending` (hasn't been issued yet)
- `Failed` (cannot cancel failed invoices)
- `Cancelled` (already cancelled)
- `Replaced` (already replaced by new invoice)

**Implementation**:
```csharp
public class InvoiceCanBeCancelledRule : IBusinessRule
{
    private readonly InvoiceReference _invoice;

    public bool IsBroken()
    {
        if (_invoice.Status != InvoiceStatus.Approved && 
            _invoice.Status != InvoiceStatus.Issued)
            return true;

        if (_invoice.Status == InvoiceStatus.Issued &&
            _invoice.IssuedAt.HasValue &&
            (DateTime.UtcNow - _invoice.IssuedAt.Value).TotalHours > 24)
            return true;

        return false;
    }

    public string Message => 
        $"Invoice in status {_invoice.Status} cannot be cancelled";
}
```

---

### BR-I-E-005: Webhook Signature Validation
**Rule**: Invoice status updates via webhook must be authenticated

**Validation**:
- HMAC-SHA256 signature verification
- Timestamp validation (prevent replay attacks)
- Source IP validation (optional)

**Implementation**:
```csharp
public class WebhookSignatureValidator
{
    public async Task<bool> ValidateAsync(HttpRequest request, string secret)
    {
        var providedSignature = request.Headers["X-Signature"].FirstOrDefault();
        var timestamp = request.Headers["X-Timestamp"].FirstOrDefault();
        var payload = await new StreamReader(request.Body).ReadToEndAsync();

        // Check timestamp (±5 minutes)
        if (!long.TryParse(timestamp, out var ts) ||
            Math.Abs(DateTimeOffset.UtcNow.ToUnixTimeSeconds() - ts) > 300)
            return false;

        // Verify signature
        var message = $"{timestamp}.{payload}";
        var expectedSignature = ComputeHmac(message, secret);

        return providedSignature == expectedSignature;
    }

    private string ComputeHmac(string message, string secret)
    {
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(message));
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }
}
```

---

### BR-I-E-006: Invoice Amount Validation
**Rule**: Invoice amounts must match transaction amounts

**Validation**:
- Total amount must equal sum of line items
- Tax amount must be correctly calculated
- Discount amounts must be properly applied
- Rounding must follow Vietnam regulations

**Implementation**:
```csharp
public class InvoiceAmountValidationRule : IBusinessRule
{
    private readonly InvoiceCreationRequest _request;

    public bool IsBroken()
    {
        var calculatedTotal = _request.LineItems
            .Sum(item => item.TotalAmount);

        var calculatedTax = _request.LineItems
            .Sum(item => item.TaxAmount);

        var calculatedGrandTotal = calculatedTotal + calculatedTax - _request.Payment.DiscountAmount;

        return Math.Abs(calculatedGrandTotal - _request.Payment.TotalAmount) > 0.01m ||
               Math.Abs(calculatedTax - _request.Payment.TaxAmount) > 0.01m;
    }

    public string Message => 
        "Invoice amounts don't match calculated totals";
}
```

---

### BR-I-E-007: Provider API Rate Limiting
**Rule**: Respect provider API rate limits to prevent blocking

**Rate Limits** (example for WiInvoice):
- 100 requests per minute
- 1000 requests per hour
- Automatic retry with exponential backoff

**Implementation**:
```csharp
public class RateLimitingHandler : DelegatingHandler
{
    private readonly SemaphoreSlim _semaphore = new(10); // Max 10 concurrent
    private readonly DateTime _windowStart = DateTime.UtcNow;
    private int _requestCount = 0;
    private const int MaxRequestsPerMinute = 100;

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        await _semaphore.WaitAsync(cancellationToken);
        try
        {
            // Reset counter every minute
            if ((DateTime.UtcNow - _windowStart).TotalMinutes >= 1)
            {
                _requestCount = 0;
            }

            if (_requestCount >= MaxRequestsPerMinute)
            {
                await Task.Delay(TimeSpan.FromSeconds(60), cancellationToken);
                _requestCount = 0;
            }

            _requestCount++;
            return await base.SendAsync(request, cancellationToken);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice Model](./model.md)
- [Invoice Lifecycle](./lifecycle.md)
- [Electronic Invoice Integration](./electronic-invoice-integration.md)
- [Domain Events](./domain-events.md)
