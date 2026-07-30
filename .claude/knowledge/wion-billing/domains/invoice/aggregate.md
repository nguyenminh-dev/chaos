# Invoice Aggregate

## Purpose
Map payments to electronic invoices through provider integration for subscription billing, platform services, and customer billing.

## Aggregate Root
**`InvoiceReference`**

## Aggregate Definition
```csharp
public class InvoiceReference : BillingAggregateRoot
{
    // Core identity
    public string Id { get; private set; }
    public string PaymentId { get; private set; }
    public string TenantId { get; private set; }
    
    // Invoice identification
    public string InvoiceNumber { get; private set; }
    public string ProviderInvoiceId { get; private set; }
    public string InvoiceUrl { get; private set; }
    
    // Classification
    public InvoiceType InvoiceType { get; private set; }
    public InvoiceStatus Status { get; private set; }
    
    // Financial data
    public decimal Amount { get; private set; }
    public decimal TaxAmount { get; private set; }
    public decimal TotalAmount { get; private set; }
    
    // Timestamps
    public DateTime? IssuedAt { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }
    
    // Processing metadata
    public int RetryCount { get; private set; }
    public string? ErrorMessage { get; private set; }
    public bool RequiresSync { get; private set; }
    
    // Additional data
    public Metadata Metadata { get; private set; }
}
```

## Business Invariants

### Core Invariants
- **Invoice Uniqueness**: One invoice per payment/transaction
- **Invoice Number Format**: Must follow Vietnam electronic invoice regulations
- **Status Transition**: Must follow valid state machine transitions
- **Retry Limits**: Failed invoice creation max 3 retry attempts
- **Provider Constraints**: Must have valid provider configuration

### Electronic Invoice Invariants
- **Provider Integration**: Requires active provider configuration
- **Authentication**: Provider API calls must be authenticated with valid signatures
- **Data Validation**: Invoice data must pass provider validation before submission
- **Webhook Handling**: Status updates must be validated for authenticity
- **Idempotency**: Same payment cannot generate multiple invoices

## Domain Events

### InvoiceGeneratedDomainEvent
```csharp
public class InvoiceGeneratedDomainEvent : IDomainEvent
{
    public string InvoiceId { get; init; }
    public string InvoiceNumber { get; init; }
    public string PaymentId { get; init; }
    public decimal TotalAmount { get; init; }
    public DateTime OccurredOn { get; init; }
}
```
**Triggered**: When invoice is successfully submitted to provider

### InvoiceIssuedDomainEvent
```csharp
public class InvoiceIssuedDomainEvent : IDomainEvent
{
    public string InvoiceId { get; init; }
    public string InvoiceNumber { get; init; }
    public string ProviderInvoiceId { get; init; }
    public string TrackingUrl { get; init; }
    public DateTime OccurredOn { get; init; }
}
```
**Triggered**: When invoice is approved and issued by tax authority

### InvoiceFailedDomainEvent
```csharp
public class InvoiceFailedDomainEvent : IDomainEvent
{
    public string InvoiceId { get; init; }
    public string PaymentId { get; init; }
    public string ErrorMessage { get; init; }
    public int RetryCount { get; init; }
    public bool CanRetry { get; init; }
    public DateTime OccurredOn { get; init; }
}
```
**Triggered**: When invoice generation fails after all retries

### InvoiceStatusChangedDomainEvent
```csharp
public class InvoiceStatusChangedDomainEvent : IDomainEvent
{
    public string InvoiceId { get; init; }
    public InvoiceStatus OldStatus { get; init; }
    public InvoiceStatus NewStatus { get; init; }
    public string ProviderInvoiceId { get; init; }
    public DateTime OccurredOn { get; init; }
}
```
**Triggered**: When invoice status changes via webhook sync

### InvoiceCancelledDomainEvent
```csharp
public class InvoiceCancelledDomainEvent : IDomainEvent
{
    public string InvoiceId { get; init; }
    public string InvoiceNumber { get; init; }
    public string ProviderInvoiceId { get; init; }
    public string Reason { get; init; }
    public DateTime OccurredOn { get; init; }
}
```
**Triggered**: When invoice is successfully cancelled

## Transaction Boundary
**Single invoice reference per transaction**

## Aggregate Operations

### Creation
```csharp
public static InvoiceReference Create(
    string paymentId,
    string invoiceNumber,
    InvoiceType invoiceType,
    decimal amount,
    decimal taxAmount)
{
    var invoice = new InvoiceReference
    {
        Id = GuidGenerator.Create(),
        PaymentId = paymentId,
        InvoiceNumber = invoiceNumber,
        InvoiceType = invoiceType,
        Amount = amount,
        TaxAmount = taxAmount,
        TotalAmount = amount + taxAmount,
        Status = InvoiceStatus.Pending,
        RetryCount = 0,
        RequiresSync = true,
        CreatedAt = DateTime.UtcNow
    };

    invoice.AddLocalEvent(new InvoiceGeneratedDomainEvent
    {
        InvoiceId = invoice.Id,
        InvoiceNumber = invoice.InvoiceNumber,
        PaymentId = invoice.PaymentId,
        TotalAmount = invoice.TotalAmount,
        OccurredOn = DateTime.UtcNow
    });

    return invoice;
}
```

### Issue Invoice
```csharp
public void Issue(string providerInvoiceId, string invoiceUrl)
{
    CheckRule(new InvoiceCanBeIssuedRule(this));

    ProviderInvoiceId = providerInvoiceId;
    InvoiceUrl = invoiceUrl;
    Status = InvoiceStatus.Issued;
    IssuedAt = DateTime.UtcNow;
    UpdatedAt = DateTime.UtcNow;

    AddLocalEvent(new InvoiceIssuedDomainEvent
    {
        InvoiceId = Id,
        InvoiceNumber = InvoiceNumber,
        ProviderInvoiceId = ProviderInvoiceId,
        TrackingUrl = InvoiceUrl,
        OccurredOn = DateTime.UtcNow
    });
}
```

### Mark Failed
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
    UpdatedAt = DateTime.UtcNow;

    AddLocalEvent(new InvoiceFailedDomainEvent
    {
        InvoiceId = Id,
        PaymentId = PaymentId,
        ErrorMessage = errorMessage,
        RetryCount = RetryCount,
        CanRetry = RetryCount < MaxRetryAttempts,
        OccurredOn = DateTime.UtcNow
    });
}
```

### Update Status (Webhook)
```csharp
public void UpdateStatus(InvoiceStatus newStatus, string? providerStatusName = null)
{
    var oldStatus = Status;
    
    CheckRule(new InvoiceStatusTransitionRule(this, newStatus));

    Status = newStatus;
    UpdatedAt = DateTime.UtcNow;

    if (newStatus == InvoiceStatus.Issued)
    {
        IssuedAt = DateTime.UtcNow;
    }

    AddLocalEvent(new InvoiceStatusChangedDomainEvent
    {
        InvoiceId = Id,
        OldStatus = oldStatus,
        NewStatus = newStatus,
        ProviderInvoiceId = ProviderInvoiceId,
        OccurredOn = DateTime.UtcNow
    });
}
```

### Cancel
```csharp
public void Cancel(string reason)
{
    CheckRule(new InvoiceCanBeCancelledRule(this));

    Status = InvoiceStatus.Cancelled;
    UpdatedAt = DateTime.UtcNow;
    RequiresSync = false;

    AddLocalEvent(new InvoiceCancelledDomainEvent
    {
        InvoiceId = Id,
        InvoiceNumber = InvoiceNumber,
        ProviderInvoiceId = ProviderInvoiceId,
        Reason = reason,
        OccurredOn = DateTime.UtcNow
    });
}
```

## Related Documents
- [Invoice Model](./model.md)
- [Invoice Business Rules](./business-rules.md)
- [Invoice Lifecycle](./lifecycle.md)
- [Electronic Invoice Integration](./electronic-invoice-integration.md)
- [Domain Events](./domain-events.md)
