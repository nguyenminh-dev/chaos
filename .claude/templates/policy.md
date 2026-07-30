# Policy Documentation Template

**Purpose**: Template for documenting cross-aggregate coordination policies.

**When to Use**: When documenting business logic that spans multiple aggregates.

---

## Policy Structure

Every policy must follow this structure:

```markdown
# {Policy Name}

## Purpose
{What this policy coordinates and why}

## Trigger
{What event or condition starts this policy}

## Participating Aggregates
- [{Aggregate1}](../domains/{aggregate1}/aggregate.md) - {Role}
- [{Aggregate2}](../domains/{aggregate2}/aggregate.md) - {Role}
- [{Aggregate3}](../domains/{aggregate3}/aggregate.md) - {Role}

## Domain Events

### Consumes
- [{Event1}](../domains/{domain1}/domain-events.md#{event}) - {Why it's consumed}

### Publishes
- [{Event2}](../domains/{domain2}/domain-events.md#{event}) - {Why it's published}
- [{Event3}](../domains/{domain3}/domain-events.md#{event}) - {Why it's published}

## Flow

### Step 1: {Step Name}
{What happens in this step}
- **Aggregate**: {Which aggregate is involved}
- **Operation**: {What operation is performed}
- **Validation**: {What business rules are checked}

### Step 2: {Step Name}
{What happens in this step}
- **Aggregate**: {Which aggregate is involved}
- **Operation**: {What operation is performed}
- **Validation**: {What business rules are checked}

### Step 3: {Step Name}
{What happens in this step}
- **Aggregate**: {Which aggregate is involved}
- **Operation**: {What operation is performed}
- **Validation**: {What business rules are checked}

## Alternative Flows

### {Alternative Scenario}
{What triggers this alternative flow}
1. {Alternative step 1}
2. {Alternative step 2}

## Failure Handling

### {Failure Scenario}
{What can go wrong}
1. {Compensation action 1}
2. {Compensation action 2}
3. {Rollback mechanism}

## Business Rules

### Rule Enforcement
This policy coordinates but does not define business rules. Business rules are enforced by individual aggregates:

- [{Aggregate1} Business Rules](../domains/{aggregate1}/business-rules.md) - {Which rules apply}
- [{Aggregate2} Business Rules](../domains/{aggregate2}/business-rules.md) - {Which rules apply}

## Implementation

### Policy Handler
```csharp
public class {Policy}Handler : ILocalEventHandler<{TriggerEvent}>
{
    public async Task HandleEventAsync({TriggerEvent} eventData)
    {
        // Policy implementation
    }
}
```

## Testing

### Policy Test Pattern
```csharp
[Fact]
public async Task Should_Coordinate_{Aggregates}_When_{Trigger}()
{
    // Arrange
    var {triggerEvent} = new {TriggerEvent}(...);
    
    // Act
    await _policyHandler.HandleEventAsync({triggerEvent});
    
    // Assert
    // Verify all aggregates updated correctly
}
```

## Related Documents
- [{UseCase1}](../application/use-cases/{use-case1}.md) - {Relationship}
- [{UseCase2}](../application/use-cases/{use-case2}.md) - {Relationship}
```

---

## Critical Principles

### 1. Policies Coordinate, Don't Define Rules

**❌ WRONG**: Defining business rules in policies

```markdown
## Business Rules
- Payment amount must be positive ❌ WRONG
- Invoice number must be unique ❌ WRONG
```

**✅ CORRECT**: Referencing aggregate business rules

```markdown
## Business Rules
This policy coordinates enforcement of:
- [Payment Business Rules](../domains/payment/business-rules.md)
- [Invoice Business Rules](../domains/invoice/business-rules.md)
```

### 2. Policies Handle Cross-Aggregate Logic

**What belongs in Policies**:
- ✅ Coordinating multiple aggregates
- ✅ Event-driven workflows
- ✅ Compensation actions
- ✅ Eventual consistency

**What belongs in Aggregates**:
- ❌ Business rules (in aggregate only)
- ❌ Domain operations (in aggregate only)
- ❌ Single aggregate logic (in aggregate only)

### 3. Event-Driven Coordination

**Pattern**: Policy subscribes to domain event → Coordinates aggregates → Publishes new events

```
{Event1} → [Policy] → {Aggregate1} + {Aggregate2} → {Event2}
```

---

## Example: Invoice After Payment Policy

```markdown
# Invoice After Payment Policy

## Purpose
Coordinate automatic invoice generation when payment succeeds

## Trigger
PaymentSucceededDomainEvent from Payment Aggregate

## Participating Aggregates
- [Payment](../domains/payment/aggregate.md) - Event source
- [InvoiceReference](../domains/invoice/aggregate.md) - Invoice creation
- [Ledger](../domains/ledger/aggregate.md) - Accounting record

## Domain Events

### Consumes
- [PaymentSucceededDomainEvent](../domains/payment/domain-events.md#payment-succeeded) - Triggers invoice creation

### Publishes
- [InvoiceIssuedDomainEvent](../domains/invoice/domain-events.md#invoice-issued) - On successful invoice creation
- [InvoiceFailedDomainEvent](../domains/invoice/domain-events.md#invoice-failed) - On invoice creation failure

## Flow

### Step 1: Receive Payment Success Event
- **Event**: PaymentSucceededDomainEvent
- **Data**: PaymentId, Amount, TenantId
- **Validation**: Verify payment is in COMPLETED status

### Step 2: Create Invoice Reference
- **Aggregate**: InvoiceReference
- **Operation**: CreateNew(paymentId, amount, invoiceType)
- **Validation**: 
  - [Invoice Business Rules](../domains/invoice/business-rules.md)
  - Specifically: BR-I-001 (One invoice per payment)

### Step 3: Call Invoice Hub API
- **Service**: External Invoice Hub
- **Operation**: Generate electronic invoice
- **Data**: Invoice details, payment information

### Step 4: Update Invoice Reference
- **Aggregate**: InvoiceReference
- **Operation**: Issue(invoiceNumber, hubId, url)
- **Validation**: Invoice transition from PENDING to ISSUED

### Step 5: Create Ledger Entry
- **Aggregate**: LedgerEntry
- **Operation**: CreateDoubleEntry(debitAccount, creditAccount, amount)
- **Validation**: 
  - [Ledger Business Rules](../domains/ledger/business-rules.md)
  - Specifically: BR-L-001 (Double-entry balance)

### Step 6: Publish Invoice Issued Event
- **Event**: InvoiceIssuedDomainEvent
- **Consumers**: Notification service, analytics service

## Alternative Flows

### Retry on Invoice Hub Failure
If Invoice Hub API fails:
1. Wait for exponential backoff (1s, 2s, 4s, 8s, 16s)
2. Retry invoice generation up to 5 times
3. Mark invoice as FAILED if all retries exhausted

## Failure Handling

### Invoice Hub Unavailable
1. Mark invoice reference as FAILED
2. Publish InvoiceFailedDomainEvent
3. Schedule retry job for 5 minutes later
4. Alert operations team

### Invalid Invoice Data
1. Mark invoice reference as FAILED
2. Publish InvoiceFailedDomainEvent with validation errors
3. Notify finance team for manual intervention

### Ledger Creation Failure
1. Rollback invoice to PENDING status
2. Publish InvoiceFailedDomainEvent
3. Alert operations team

## Business Rules

### Rule Enforcement
This policy coordinates enforcement of:
- [Invoice Business Rules](../domains/invoice/business-rules.md)
  - Specifically: BR-I-001 (One invoice per payment)
- [Ledger Business Rules](../domains/ledger/business-rules.md)
  - Specifically: BR-L-001 (Double-entry balance)

**Note**: Business rules are enforced by individual aggregates, not by this policy.

## Implementation

### Policy Handler
```csharp
public class InvoiceAfterPaymentPolicy : ILocalEventHandler<PaymentSucceededDomainEvent>
{
    private readonly IInvoiceReferenceRepository _invoiceRepository;
    private readonly ILedgerEntryRepository _ledgerRepository;
    private readonly IInvoiceHubClient _invoiceHubClient;
    
    public async Task HandleEventAsync(PaymentSucceededDomainEvent eventData)
    {
        // Step 1: Create invoice reference
        var invoice = InvoiceReference.CreateNew(
            eventData.PaymentId,
            eventData.TenantId,
            InvoiceType.BanHang,
            eventData.Amount,
            CalculateTax(eventData.Amount)
        );
        
        await _invoiceRepository.InsertAsync(invoice);
        
        // Step 2: Call Invoice Hub with retry
        try
        {
            var invoiceResult = await _invoiceHubClient.GenerateInvoiceAsync(invoice);
            
            // Step 3: Update invoice as issued
            invoice.Issue(
                invoiceResult.InvoiceNumber,
                invoiceResult.HubId,
                invoiceResult.InvoiceUrl
            );
            
            await _invoiceRepository.UpdateAsync(invoice);
            
            // Step 4: Create ledger entry
            var ledgerEntry = LedgerEntry.CreateDoubleEntry(
                $"PAYMENT-{eventData.PaymentId}",
                eventData.Amount,
                "DEBIT",
                "CREDIT"
            );
            
            await _ledgerRepository.InsertAsync(ledgerEntry);
        }
        catch (InvoiceHubException ex)
        {
            // Failure handling
            invoice.Fail($"Invoice Hub failed: {ex.Message}");
            await _invoiceRepository.UpdateAsync(invoice);
        }
    }
}
```

## Testing

### Policy Test Pattern
```csharp
public class InvoiceAfterPaymentPolicyTests
{
    [Fact]
    public async Task Should_Create_Invoice_When_Payment_Succeeds()
    {
        // Arrange
        var paymentEvent = new PaymentSucceededDomainEvent(
            paymentId: 123,
            tenantId: "tenant-123",
            amount: 100000m
        );
        
        // Act
        await _policyHandler.HandleEventAsync(paymentEvent);
        
        // Assert
        var invoice = await _invoiceRepository.FindByPaymentIdAsync(123);
        invoice.ShouldNotBeNull();
        invoice.Status.ShouldBe(InvoiceStatus.Issued);
    }
    
    [Fact]
    public async Task Should_Handle_Invoice_Hub_Failure()
    {
        // Arrange
        var paymentEvent = new PaymentSucceededDomainEvent(...);
        _invoiceHubClient.Setup(x => x.GenerateInvoiceAsync(It.IsAny<Invoice>()))
            .ThrowsAsync(new InvoiceHubException("Service unavailable"));
        
        // Act
        await _policyHandler.HandleEventAsync(paymentEvent);
        
        // Assert
        var invoice = await _invoiceRepository.FindByPaymentIdAsync(123);
        invoice.Status.ShouldBe(InvoiceStatus.Failed);
    }
}
```

## Related Documents
- [Create Invoice Use Case](../application/use-cases/create-invoice.md) - Manual invoice creation
- [Process Payment Use Case](../application/use-cases/process-payment.md) - Payment processing
```

---

## Common Policy Patterns

### 1. Event Chaining Pattern

**Purpose**: Chain multiple aggregates in response to one event

```
Event1 → Policy → Aggregate1 → Aggregate2 → Event2
```

### 2. Compensation Pattern

**Purpose**: Rollback multiple aggregates on failure

```
Event1 → Policy → Aggregate1 + Aggregate2
Failure → Policy → Rollback1 + Rollback2
```

### 3. Fan-Out Pattern

**Purpose**: Trigger multiple independent operations

```
Event1 → Policy → Aggregate1
              → Aggregate2
              → Aggregate3
```

### 4. Retry Pattern

**Purpose**: Handle transient failures

```
Event1 → Policy → Try Operation
       Failure → Wait → Retry
       Failure → Wait → Retry
       Failure → Mark Failed
```

---

## Policy vs Aggregate Decision Guide

### When to Create a Policy

**Create a Policy when**:
- ✅ Logic spans 2+ aggregates
- ✅ Event-driven coordination needed
- ✅ Compensation actions required
- ✅ Eventual consistency acceptable

**Don't create a Policy when**:
- ❌ Logic affects single aggregate (put in aggregate)
- ❌ Business rule validation (put in aggregate)
- ❌ Simple CRUD operation (put in application service)

---

## Policy Documentation Checklist

Before marking Policy documentation as complete:

- [ ] Purpose clearly defined
- [ ] Trigger event identified
- [ ] All participating aggregates listed
- [ ] Domain events documented
- [ ] Flow steps clearly defined
- [ ] Alternative flows documented
- [ ] Failure handling documented
- [ ] Business rules referenced (not defined)
- [ ] Implementation example provided
- [ ] Test pattern provided
- [ ] Related use cases linked
- [ ] All links resolve correctly

---

## Quick Reference

### "What goes in a Policy?"

| Section | Content | Source |
|---------|---------|--------|
| **Coordination Logic** | Multi-aggregate flow | Policy (own this) |
| **Event Flow** | Event routing | Policy (own this) |
| **Failure Handling** | Compensation actions | Policy (own this) |
| **Business Rules** | Which rules apply | Aggregates (reference) |
| **Aggregate Operations** | What operations called | Aggregates (reference) |

---

**Template Version**: 1.0  
**Last Updated**: 2026-07-27  
**Based On**: DDD Conventions and Wion Engineering Rules