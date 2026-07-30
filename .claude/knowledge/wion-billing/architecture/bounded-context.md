# Billing Service Bounded Context

## Definition
Billing Service is a **Bounded Context** within the WION platform's Financial Operations domain, responsible for all monetary transactions, credit management, wallet operations, and financial reporting.

## Context Ownership

### Owner
**Team**: Financial Operations Team
**Product Owner**: Financial Operations Product Manager
**Tech Lead**: Billing Service Technical Lead

### Stakeholders
- **Upstream Services**: Tenant Service (tenant lifecycle)
- **Downstream Services**: All WION products (POS, FnB, SPA, WIPIX, AI Services, Platform Services)
- **External Partners**: TPayGate (Payment Gateway), Invoice Hub (Electronic Invoicing)

## Context Responsibilities

### What Billing Service OWNS
1. **Wallet Management** - Tenant wallets, balance tracking, multi-asset support
2. **Payment Processing** - QR code payments, payment tracking, webhook handling
3. **Credit Operations** - Credit consumption, refunds, adjustments
4. **Invoice Generation** - Electronic invoices, regulatory compliance
5. **Ledger & Audit** - Double-entry accounting, transaction audit trail

### What Billing Service DOES NOT OWN
- ❌ Tenant lifecycle (owned by Tenant Service)
- ❌ Product business logic (owned by respective products)
- ❌ Payment gateway infrastructure (TPayGate)
- ❌ Electronic invoice infrastructure (Invoice Hub)

## Ubiquitous Language

### Core Concepts
| Term | Definition |
|------|------------|
| **Wallet** | Digital wallet for tenant funds |
| **Wi Credit** | Internal payment unit (1 VNĐ = 1 Wi Credit) |
| **Balance** | Available funds for consumption |
| **Reserved Balance** | Funds locked for pending transactions |
| **Payment** | Financial transaction for wallet topup |
| **Invoice** | Electronic invoice document |
| **Ledger** | Double-entry accounting record |

### Technical Terms
| Term | Definition |
|------|------------|
| **Aggregate** | Cluster of domain objects treated as unit |
| **Domain Event** | State change notification |
| **Use Case** | Application workflow orchestrating Aggregates |
| **Policy** | Cross-aggregate business logic coordination |

## Context Boundaries

### Inbound Relationships (Upstream)
```
Tenant Service
    ↓
[TenantCreated Event]
    ↓
Billing Service (creates wallet)
```

**Relationship**: Billing Service LISTENS to Tenant Service events

### Outbound Relationships (Downstream)
```
Billing Service
    ↓
[WION POS, FnB, SPA, WIPIX, AI Services, Platform Services]
    ↓
[Credit Consumption APIs, Webhook Callbacks]
```

**Relationship**: Downstream services CALL Billing Service APIs and RECEIVE events

### External Integrations
```
Billing Service
    ↓
[TPayGate]
    ↓
[Create Payment, QR Code Generation, Webhook Callback]

Billing Service
    ↓
[Invoice Hub]
    ↓
[Create Invoice, Issue Electronic Invoice]
```

**Relationship**: Billing Service CALLS external services, LISTENS to callbacks

## Functional Boundaries

### Within Scope
- ✅ Wallet lifecycle management
- ✅ Payment initiation and tracking
- ✅ Credit consumption and refunds
- ✅ Invoice generation and tracking
- ✅ Ledger and audit trail
- ✅ Multi-asset wallet support
- ✅ Balance validation and enforcement
- ✅ Idempotency guarantees
- ✅ Double-entry accounting

### Out of Scope
- ❌ Tenant registration/deletion (triggered by Tenant Service)
- ❌ Product-specific business logic
- ❌ Payment gateway infrastructure (TPayGate responsibility)
- ❌ Invoice infrastructure (Invoice Hub responsibility)
- ❌ User authentication (handled by platform)
- ❌ UI/UX design (handled by products)

## Non-Functional Requirements

### Performance
- Credit consumption: < 100ms (P95)
- Balance check: < 100ms (P95)
- Payment initiation: < 500ms (P95)
- Webhook processing: < 1s (P95)
- Throughput: 10,000 TPS for credit operations

### Availability
- SLA: 99.5% uptime
- Data retention: 7 years for ledger, 10 years for invoices

### Scalability
- Horizontal scaling: Stateless application
- Database: Read replicas for queries
- Cache: Redis cluster for balance caching
- Message Queue: RabbitMQ cluster for events

### Security
- Authentication: API Key based
- Authorization: Role-based (Admin, Finance, Tenant)
- Tenant Isolation: Row-level security
- Webhook Security: HMAC-SHA256 signature verification
- Rate Limiting: Per-tenant limits

## Related Documents
- [Context Map](./architecture/context-map.md) - Relationships with other contexts
- [Domain Overview](./domains/) - Business domain details
- [Use Cases](./application/use-cases/) - Application workflows
- [API Documentation](./api/) - Service contracts
