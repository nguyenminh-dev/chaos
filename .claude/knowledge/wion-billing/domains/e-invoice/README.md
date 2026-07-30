# E-Invoice Domain

## Purpose
E-Invoice domain manages the integration with external e-invoice providers (like WiInvoice, VNPT, Viettel, etc.) for generating compliant e-invoices for subscription billing, platform services, and customer billing operations.

## Business Context

### Wion Billing Platform vs POS Billing
**Wion Billing Platform** handles:
- **Subscription billing**: Recurring charges for platform services
- **Payment processing**: Online payment transactions
- **Renewals**: Subscription renewals and upgrades
- **Platform services**: API usage, storage, compute, AI services
- **Customer billing**: Direct customer invoicing

**POS Billing** (billing-management) handles:
- Point-of-sale transactions
- Retail operations
- In-person customer interactions
- Shop-specific configurations

### Key Differences
| Aspect | Wion Billing | POS Billing |
|--------|-------------|-------------|
| **Primary Domain** | Platform Services & Subscriptions | Retail & POS Operations |
| **Invoice Trigger** | Subscription cycles, payment completion | Order completion |
| **Customer** | Platform tenants, end users, organizations | In-store customers |
| **Pricing** | Subscription plans, usage-based | One-time purchases |
| **Tax Model** | Service tax, digital services | VAT on goods |
| **Volume** | Lower volume, higher complexity | High volume, standardized |
| **Integration** | Payment gateways, subscription systems | POS terminals, e-commerce |

---

## Business Requirements

### Functional Requirements

#### FR1: Provider Integration
- SHALL support multiple e-invoice providers
- SHALL abstract provider-specific implementations behind interfaces
- SHALL handle provider authentication and authorization
- SHALL manage provider-specific configurations per tenant

#### FR2: Invoice Generation
- SHALL generate e-invoices for completed payments
- SHALL support subscription billing invoices
- SHALL support usage-based billing invoices
- SHALL support one-time service invoices
- SHALL handle invoice cancellation and replacement

#### FR3: Invoice Status Management
- SHALL track invoice lifecycle statuses
- SHALL synchronize status with external providers
- SHALL handle webhook callbacks for status updates
- SHALL maintain audit trail of status changes

#### FR4: Compliance and Validation
- SHALL validate invoice data before submission
- SHALL ensure tax compliance for digital services
- SHALL support multiple invoice templates
- SHALL handle customer billing profile information

### Non-Functional Requirements

#### NFR1: Scalability
- SHALL handle concurrent invoice generation requests
- SHALL support high-volume invoice processing
- SHALL implement background job processing

#### NFR2: Reliability
- SHALL ensure exactly-once processing semantics
- SHALL prevent duplicate invoice generation
- SHALL handle network failures gracefully

#### NFR3: Security
- SHALL secure provider credentials
- SHALL encrypt sensitive customer data
- SHALL implement signature validation for webhooks
- SHALL maintain audit logs

#### NFR4: Performance
- SHALL process invoices asynchronously
- SHALL provide real-time status updates
- SHALL minimize external API call latency

---

## Domain Model

### Core Entities

#### EInvoice (Aggregate Root)
```csharp
// Represents a single e-invoice in the system
- Id: Guid
- InvoiceNumber: string (Unique per tenant)
- ReferenceId: string (External reference - subscription_id, payment_id)
- ReferenceType: InvoiceReferenceType (Subscription, Payment, Manual)
- Status: InvoiceStatus
- ProviderInvoiceId: string? (External provider invoice ID)
- ProviderTrackingUrl: string?
- Customer: CustomerInfo
- Seller: SellerInfo
- LineItems: List<InvoiceLineItem>
- Totals: InvoiceTotals
- TaxSummary: TaxSummary
- IssuedDate: DateTime?
- ApprovedDate: DateTime?
- Currency: string
- TemplateCode: string?
- BillingProfile: BillingProfileInfo?
- ErrorMessage: string?
- CreatedAt: DateTime
- UpdatedAt: DateTime
```

#### EInvoiceSettings (Aggregate Root)
```csharp
// Provider configuration per tenant
- Id: Guid
- TenantId: string
- Provider: InvoiceProvider
- ProviderConfigId: string (External provider config ID)
- ProviderApiKey: string (Encrypted)
- ProviderApiSecret: string (Encrypted)
- ProviderTaxCode: string
- DefaultTemplateCode: string?
- WebhookUrl: string?
- AutoIssueEnabled: bool
- IsActive: bool
- ValidFrom: DateTime?
- ValidTo: DateTime?
- CreatedAt: DateTime
- UpdatedAt: DateTime
```

#### ElectronicInvoiceProvider (Value Object + Enum)
```csharp
// Provider type enum
enum InvoiceProvider
{
    WiInvoice,
    VNPT,
    Viettel,
    MISA,
    Other
}

// Provider capabilities value object
class ProviderCapabilities
{
    bool SupportsCancellation { get; }
    bool SupportsReplacement { get; }
    bool SupportsBatchOperations { get; }
    bool SupportsWebhookCallbacks { get; }
    int MaxRetryAttempts { get; }
}
```

### Supporting Entities

#### InvoiceLineItem (Entity)
```csharp
- Id: Guid
- InvoiceId: Guid
- Code: string? (Service code)
- Name: string
- Description: string?
- Quantity: decimal
- UnitPrice: decimal
    - DiscountAmount: decimal
- TaxRate: decimal (Percentage)
- TaxAmount: decimal
- TotalAmount: decimal (UnitPrice * Quantity + TaxAmount)
- ItemType: LineItemType (Service, Usage, Discount)
- ServicePeriod: DateRange?
```

#### CustomerInfo (Value Object)
```csharp
- Type: CustomerType (Individual, Business)
- DisplayName: string? (For individuals)
- LegalName: string? (For businesses)
- TaxCode: string? (Tax ID)
- Email: string?
- Phone: string?
- Address: string?
- IdentityCard: string? (ID number for individuals)
```

#### SellerInfo (Value Object)
```csharp
- Name: string
- TaxCode: string
- Address: string
- Phone: string?
- Email: string?
```

### Enums

#### InvoiceStatus
```csharp
enum InvoiceStatus
{
    Draft = 0,           // Initial state
    Pending = 1,         // Submitted to provider
    Processing = 2,      // Provider is processing
    Approved = 3,        // Approved by tax authority
    Issued = 4,         // Final issued state
    Cancelled = 5,       // Cancelled
    Replaced = 6,        // Replaced by new invoice
    Failed = 7,          // Generation failed
    RetryPending = 8     // Waiting for retry
}
```

#### InvoiceReferenceType
```csharp
enum InvoiceReferenceType
{
    Subscription,     // Recurring subscription invoice
    Payment,          // One-time payment invoice
    Manual,           // Manually created invoice
    UsageBased,       // Usage-based billing invoice
    Credit            // Credit adjustment invoice
}
```

---

## Business Rules

### BR-001: Invoice Uniqueness
**Rule**: Each reference ID can have only one active electronic invoice.

**Validation**:
```csharp
public class InvoiceMustBeUniqueRule : IBusinessRule
{
    private readonly string _referenceId;
    private readonly IInvoiceReferenceRepository _repository;

    public bool IsBroken()
    {
        var existing = _repository.FindByReferenceId(_referenceId);
        return existing != null && existing.Status != InvoiceStatus.Cancelled;
    }

    public string Message => $"Invoice for reference {_referenceId} already exists";
}
```

### BR-002: Invoice Data Validation
**Rule**: Invoice must have valid customer information, line items, and totals before submission.

**Validation**:
```csharp
public class InvoiceMustHaveValidDataRule : IBusinessRule
{
    public bool IsBroken()
    {
        return string.IsNullOrEmpty(Customer.TaxCode) && 
               string.IsNullOrEmpty(Customer.Email);
    }

    public string Message => "Customer must have either tax code or email";
}
```

### BR-003: Provider Configuration
**Rule**: Invoice generation requires active provider configuration.

**Validation**:
```csharp
public class ProviderMustBeConfiguredRule : IBusinessRule
{
    public bool IsBroken()
    {
        var settings = _settingsRepository.FindActive(_tenantId);
        return settings == null || !settings.IsActive;
    }

    public string Message => "E-invoice provider not configured for tenant";
}
```

### BR-004: Invoice Cancellation
**Rule**: Only invoices in specific statuses can be cancelled.

**Validation**:
```csharp
public class InvoiceCanBeCancelledRule : IBusinessRule
{
    public bool IsBroken()
    {
        return _invoice.Status != InvoiceStatus.Approved && 
               _invoice.Status != InvoiceStatus.Issued;
    }

    public string Message => "Only approved or issued invoices can be cancelled";
}
```

---

## Domain Events

### InvoiceGeneratedDomainEvent
```csharp
public class InvoiceGeneratedDomainEvent : IDomainEvent
{
    Guid InvoiceId { get; }
    string InvoiceNumber { get; }
    string ReferenceId { get; }
    decimal TotalAmount { get; }
    DateTime OccurredOn { get; }
}
```

### InvoiceStatusChangedDomainEvent
```csharp
public class InvoiceStatusChangedDomainEvent : IDomainEvent
{
    Guid InvoiceId { get; }
    InvoiceStatus OldStatus { get; }
    InvoiceStatus NewStatus { get; }
    string ProviderInvoiceId { get; }
    DateTime OccurredOn { get; }
}
```

### InvoiceFailedDomainEvent
```csharp
public class InvoiceFailedDomainEvent : IDusinessEvent
{
    Guid InvoiceId { get; }
    string ReferenceId { get; }
    string ErrorMessage { get; }
    DateTime OccurredOn { get; }
}
```

### InvoiceApprovedDomainEvent
```csharp
public class InvoiceApprovedDomainEvent : IDomainEvent
{
    Guid InvoiceId { get; }
    string InvoiceNumber { get; }
    string ProviderInvoiceId { get; }
    string TrackingUrl { get; }
    DateTime ApprovedDate { get; }
    DateTime OccurredOn { get; }
}
```

---

## Integration Patterns

### Provider Abstraction
```csharp
public interface IEInvoiceProvider
{
    Task<InvoiceCreationResult> CreateInvoiceAsync(InvoiceCreationRequest request);
    Task<InvoiceStatusResult> GetInvoiceStatusAsync(string providerInvoiceId);
    Task<InvoiceCancellationResult> CancelInvoiceAsync(string providerInvoiceId);
    Task<InvoicePdfResult> GetInvoicePdfAsync(string providerInvoiceId);
    Task<bool> ValidateWebhookSignatureAsync(HttpContext context);
}
```

### Configuration Management
```csharp
public class EInvoiceOptions
{
    string DefaultProvider { get; set; }
    Dictionary<string, ProviderConfig> Providers { get; set; }
    bool AutoIssueEnabled { get; set; }
}
```

### Error Handling Strategy
```csharp
public class InvoiceErrorHandler
{
    async Task HandleErrorAsync(EInvoice invoice, Exception ex)
    {
        // No retry logic - mark as failed directly
        await MarkAsFailedAsync(invoice, ex.Message);
    }
}
```

---

## Repository Interfaces

### IEInvoiceRepository
```csharp
public interface IEInvoiceRepository : IRepository<EInvoice, Guid>
{
    Task<EInvoice?> FindByInvoiceNumberAsync(string invoiceNumber);
    Task<EInvoice?> FindByReferenceIdAsync(string referenceId);
    Task<EInvoice?> FindByProviderInvoiceIdAsync(string providerInvoiceId);
    Task<List<EInvoice>> FindByStatusAsync(InvoiceStatus status);
}
```

### IEInvoiceSettingsRepository
```csharp
public interface IEInvoiceSettingsRepository : IRepository<EInvoiceSettings, Guid>
{
    Task<EInvoiceSettings?> FindActiveByTenantAsync(string tenantId);
    Task<List<EInvoiceSettings>> FindByProviderAsync(InvoiceProvider provider);
}
```

---

## Use Case Connections

The E-Invoice domain integrates with:

1. **Payment Domain**: Auto-generate invoices on successful payments
2. **Subscription Domain**: Generate recurring invoices for subscriptions
3. **Credit Transaction Domain**: Invoice credit purchases
4. **Wallet Domain**: Invoice wallet top-ups

---

## Testing Strategy

### Unit Tests
- Business rule validation
- Domain state transitions
- Domain event generation
- Value object equality

### Integration Tests
- Provider API integration
- Repository operations
- Event publishing
- Webhook handling

### Acceptance Tests
- End-to-end invoice generation
- Multi-provider scenarios
- Error recovery flows
- Performance under load

---

## Next Steps
- [Provider Abstraction](./provider-abstraction.md)
- [Domain Operations](./operations.md)
- [Integration Events](./events.md)
- [API Contracts](../api/e-invoice.md)
