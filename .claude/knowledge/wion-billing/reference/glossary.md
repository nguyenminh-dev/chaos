# Billing Service - Glossary

This glossary contains Billing Service-specific terminology. For universal DDD and architecture terms, see the [Shared Glossary](../../../shared/glossary.md).

---

## Billing Service Domain Terms

### Wi Credit
**Definition**: Internal payment unit of WION Platform.

**Conversion Rate**: 1 VNĐ = 1 Wi Credit (configurable)

**Constraints**:
- Cannot reverse convert to cash
- Can be consumed across all WION products

**Related**: [Wallet Domain](../domains/wallet/overview.md)

---

### Wallet
**Definition**: Digital wallet belonging to a tenant that stores balance and assets.

**Context**: Each tenant has exactly one wallet.

**Components**:
- Available balance - Funds that can be used
- Reserved balance - Funds locked for pending transactions
- Assets - Different types of credits (WI_CREDIT, PROMOTION, GIFT, TRIAL, AI_TOKEN)

**Related**: [Wallet Domain](../domains/wallet/overview.md)

---

### Reserved Balance
**Definition**: Funds temporarily locked for pending transactions.

**Purpose**: Prevent double-spending during transaction processing.

**Behavior**: Released back to available balance on transaction completion or failure.

**Related**: [Wallet Business Rules](../domains/wallet/business-rules.md)

---

### Asset Type
**Definition**: Different types of credits stored in wallet.

**Values**:
- `WI_CREDIT`: Standard WION credits
- `PROMOTION`: Promotional credits with expiration
- `GIFT`: Gift credits from campaigns
- `TRIAL`: Trial credits for new users
- `AI_TOKEN`: AI service tokens (future)

**Constraints**: One asset per type per wallet.

**Related**: [Wallet Model](../domains/wallet/model.md)

---

## Universal DDD Terms

For universal Domain-Driven Design and Clean Architecture terminology, see:

- [Aggregate](../../../shared/glossary.md#aggregate) - Cluster of domain objects
- [Aggregate Root](../../../shared/glossary.md#aggregate-root) - Entry point for aggregate
- [Value Object](../../../shared/glossary.md#value-object) - Immutable domain concept
- [Domain Event](../../../shared/glossary.md#domain-event) - Something that happened
- [Repository](../../../shared/glossary.md#repository) - Persistence abstraction
- [Transaction Boundary](../../../shared/glossary.md#transaction-boundary) - Transaction scope

---

## Integration Terms

### TPayGate
**Definition**: Payment gateway provider for QR code payments.

**Role**: Process QR payments and send webhooks.

**Integration**: REST API + Webhooks

**Related**: [Payment Domain](../domains/payment/overview.md)

---

### Invoice Hub
**Definition**: Electronic invoice generation system.

**Role**: Issue electronic invoices per Vietnam regulations.

**Integration**: REST API

**Related**: [Invoice Domain](../domains/invoice/overview.md)

---

## Technical Terms

### P95 Latency
**Definition**: 95th percentile response time.

**Meaning**: 95% of requests complete within this time.

**Usage**: Performance SLA metric.

**Targets**:
- Balance check: < 100ms
- Credit consume: < 100ms
- Payment initiation: < 500ms

---

### Dead Letter Queue (DLQ)
**Definition**: Queue for failed events/messages.

**Purpose**: Store events that failed processing for retry or manual intervention.

**Retry**: 3 attempts with exponential backoff before DLQ.

---

## Related Documentation

- [Shared Glossary](../../../shared/glossary.md) - Universal DDD terms
- [Wallet Domain](../domains/wallet/) - Wallet terminology
- [Payment Domain](../domains/payment/) - Payment terminology
- [Architecture Overview](../architecture/README.md) - System architecture

---

**Last Updated**: 2026-07-05
**Maintained By**: Billing Service Team
