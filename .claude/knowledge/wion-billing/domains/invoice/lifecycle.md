# Invoice Lifecycle

## Overview
This document describes the complete lifecycle of electronic invoices in Wion Billing, from payment completion to final issuance and cancellation, based on electronic invoice provider integration patterns.

## Lifecycle States

```
┌──────────────┐
│   Payment    │
│  Completed   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Pending    │ ◀───────────────────────┐
│   Created    │                          │
└──────┬───────┘                          │
       │                                   │
       ▼                                   │
┌──────────────┐                          │
│  Processing  │                          │
│(Submitted to│                          │
│   Provider) │                          │
└──────┬───────┘                          │
       │                                   │
       ▼                                   │
┌──────────────┐                          │
│   Approved   │                          │
│(By Tax Auth) │                          │
└──────┬───────┘                          │
       │                                   │
       ▼                                   │
┌──────────────┐     ┌──────────────┐     │
│    Issued    │────▶│  Cancelled   │     │
└──────────────┘     └──────────────┘     │
       │                                   │
       ▼                                   │
┌──────────────┐                          │
│   Replaced   │                          │
└──────────────┘                          │
                                           │
┌──────────────┐                          │
│    Failed    │──────────────────────────┘
└──────────────┘
```

## Detailed Lifecycle Stages

### 1. Payment Completion → Invoice Pending
**Trigger**: `PaymentCompletedDomainEvent`

**Flow**:
```csharp
// Payment Domain
public class Payment : BillingAggregateRoot
{
    public void Complete()
    {
        Status = PaymentStatus.Completed;
        CompletedAt = DateTime.UtcNow;
        
        AddLocalEvent(new PaymentCompletedDomainEvent
        {
            PaymentId = Id,
            Amount = Amount,
            TenantId = TenantId,
            Customer = Customer
        });
    }
}

// Invoice Domain - Event Handler
public class PaymentCompletedHandler : IDistributedEventHandler<PaymentCompletedEvent>
{
    public async Task HandleEventAsync(PaymentCompletedEvent eventData)
    {
        var command = new GenerateInvoiceCommand
        {
            PaymentId = eventData.PaymentId,
            Amount = eventData.Amount,
            TenantId = eventData.TenantId,
            Customer = eventData.Customer
        };

        await _invoiceAppService.GenerateInvoiceAsync(command);
    }
}
```

**State Transition**:
```
Payment Completed → Invoice Created (Pending)
```

---

### 2. Invoice Pending → Processing
**Trigger**: Automatic background job or manual trigger

**Provider Integration**:
```csharp
public class WiInvoiceService
{
    public async Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request)
    {
        var wiRequest = MapToWiInvoiceRequest(request);
        var signature = GenerateSignature(wiRequest);

        _httpClient.DefaultRequestHeaders.Add("x-api-key", _options.ApiKey);
        _httpClient.DefaultRequestHeaders.Add("x-signature", signature);
        _httpClient.DefaultRequestHeaders.Add("x-tenant", CurrentTenant.Id);

        var response = await _httpClient.PostAsJsonAsync(_options.CreateInvoice, wiRequest);
        var result = await response.Content.ReadFromJsonAsync<WiInvoiceResponse<InvoiceCreationResult>>();

        return MapToInvoiceCreationResult(result);
    }
}
```

**State Transition**:
```
Pending → Processing (submitted to provider)
```

---

### 3. Processing → Approved
**Trigger**: Provider webhook callback or polling sync

**Webhook Flow**:
```csharp
[HttpPost("webhook/invoice")]
public async Task<IActionResult> HandleWebhookAsync([FromBody] InvoiceWebhookDto webhookData)
{
    // 1. Validate webhook signature
    if (!await ValidateWebhookSignatureAsync(Request))
        return Unauthorized(new { success = false, message = "Invalid signature" });

    // 2. Find and update invoice
    var invoice = await _repository.FindByProviderInvoiceIdAsync(webhookData.Key);
    invoice.UpdateStatus(webhookData.StatusCode, webhookData.StatusName);
    await _repository.UpdateAsync(invoice);

    return Ok(new { success = true });
}
```

**State Transition**:
```
Processing → Approved (approved by tax authority)
```

---

### 4. Approved → Issued
**Trigger**: Provider webhook callback (final issuance)

**Flow**:
```csharp
public void Issue(string providerInvoiceId, string invoiceUrl)
{
    CheckRule(new InvoiceCanBeIssuedRule(this));

    ProviderInvoiceId = providerInvoiceId;
    InvoiceUrl = invoiceUrl;
    Status = InvoiceStatus.Issued;
    IssuedAt = DateTime.UtcNow;

    AddLocalEvent(new InvoiceIssuedDomainEvent
    {
        InvoiceId = Id,
        InvoiceNumber = InvoiceNumber,
        ProviderInvoiceId = ProviderInvoiceId,
        TrackingUrl = InvoiceUrl
    });
}
```

**State Transition**:
```
Approved → Issued (final state)
```

---

### 5. Any State → Failed
**Trigger**: Provider API error, validation failure, timeout

**Flow**:
```csharp
public void MarkAsFailed(string errorMessage)
{
    if (RetryCount >= MaxRetryAttempts)
    {
        Status = InvoiceStatus.Failed;
        RequiresSync = false;
    }
    else
    {
        Status = InvoiceStatus.Pending;
        RequiresSync = true;
    }

    ErrorMessage = errorMessage;
    RetryCount++;
}
```

**Retry Strategy**:
```
Attempt 1: Immediate (synchronous)
Attempt 2: 30 seconds delay
Attempt 3: 60 seconds delay
After 3: Failed state, manual intervention required
```

---

### 6. Approved → Cancelled
**Trigger**: Customer cancellation request

**Flow**:
```csharp
public async Task CancelInvoiceAsync(string invoiceNumber)
{
    var invoice = await _repository.FindByInvoiceNumberAsync(invoiceNumber);
    CheckRule(new InvoiceCanBeCancelledRule(invoice));

    await _provider.CancelInvoiceAsync(invoice.ProviderInvoiceId);
    invoice.Cancel("Customer requested cancellation");
    await _repository.UpdateAsync(invoice);
}
```

**State Transition**:
```
Approved → Cancelled
Issued → Cancelled (within 24 hours)
```

---

### 7. Issued → Replaced
**Trigger**: Invoice replacement/correction

**Flow**:
```csharp
public async Task ReplaceInvoiceAsync(string oldInvoiceNumber, CreateInvoiceDto newInvoiceData)
{
    var oldInvoice = await _repository.FindByInvoiceNumberAsync(oldInvoiceNumber);
    CheckRule(new InvoiceCanBeReplacedRule(oldInvoice));

    var newInvoice = await GenerateInvoiceAsync(newInvoiceData);
    oldInvoice.MarkAsReplaced(newInvoice.InvoiceNumber);

    await _provider.ReplaceInvoiceAsync(oldInvoice.ProviderInvoiceId, newInvoice.ProviderInvoiceId);
}
```

**State Transition**:
```
Issued (old) → Replaced
New invoice created → Issued
```

## State Transitions Summary

| From | To | Trigger | Conditions |
|------|-----|---------|------------|
| Payment Completed | Pending | PaymentCompletedEvent | Payment successful |
| Pending | Processing | Provider API call | Valid data, provider configured |
| Processing | Approved | Webhook callback | Tax authority approval |
| Processing | Failed | Provider error | Transient/permanent error |
| Approved | Issued | Final issuance | PDF available |
| Approved | Cancelled | Cancellation request | Within allowed timeframe |
| Issued | Cancelled | Cancellation request | Within 24 hours |
| Issued | Replaced | Replacement request | Provider supports replacement |
| Failed | Pending | Retry attempt | < 3 retry attempts |
| Any | Failed | Permanent error | >= 3 retry attempts |

## Error Handling & Recovery

### Transient Errors
**Network Issues**, **Provider Timeouts**, **Rate Limits**

**Recovery**: Automatic retry with exponential backoff

### Permanent Errors
**Validation Failures**, **Authentication Errors**, **Provider Rejection**

**Recovery**: Manual intervention queue

## Monitoring & Observability

### Key Metrics
- **Invoice Generation Rate**: Invoices per hour/day
- **Success Rate**: % of invoices successfully issued
- **Processing Time**: Average time from Pending → Issued
- **Provider Response Time**: API call latency
- **Error Rate**: % of invoices failing after retries

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice Business Rules](./business-rules.md)
- [Electronic Invoice Integration](./electronic-invoice-integration.md)
- [Domain Events](./domain-events.md)
