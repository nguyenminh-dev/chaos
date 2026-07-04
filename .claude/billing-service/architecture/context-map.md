# Billing Service Context Map

## Purpose
This document maps Billing Service's relationships with other Bounded Contexts within the WION platform.

## Context Relationships

### Upstream Context: Tenant Service
**Relationship Type**: **Upstream/Downstream** (Tenant Service is Upstream)

**Interaction Pattern**:
```
[Tenant Service] ──(publishes)──→ [TenantCreated Event]
                                  ↓
                           [Billing Service]
                                  ↓
                           [Creates Wallet]
```

**Integration Details**:
- **Event**: `TenantCreated`
- **Action**: Billing Service creates wallet for new tenant
- **Action**: Billing Service soft-deletes wallet on `TenantDeleted` event
- **Protocol**: Domain Event via Message Queue (RabbitMQ)
- **Coupling**: Loose (event-driven)

**Ownership**:
- Tenant Context OWNS tenant lifecycle
- Billing Context OWNS wallet lifecycle

---

### Customer Contexts: WION Products

**Relationship Type**: **Downstream/Customer** (WION Products are customers)

**Products**:
1. **WION POS** - Point of Sale system
2. **WION FnB** - Food & Beverage system
3. **WION SPA** - Service Platform
4. **WIPIX** - Media Platform
5. **AI Services** - AI Generation, OCR
6. **Platform Services** - SMS, Storage, Marketplace

**Interaction Pattern**:
```
[Billing Service] ←─(calls APIs)─── [WION Products]
                              ↓
                        [Credit Consumption]
                              ↓
[Billing Service] ──(publishes)──→ [CreditConsumed Event]
                                  ↓
                           [Products process services]
```

**Integration Details**:
- **Pattern**: API Calls + Domain Events
- **APIs Used**:
  - `POST /api/v1/credit/consume` - Consume credits
  - `POST /api/v1/credit/refund` - Refund credits
  - `GET /api/v1/wallets/{tenantId}/balance` - Check balance
- **Events Published**:
  - `CreditConsumed` - After successful consumption
  - `BalanceChanged` - After balance update
  - `InsufficientBalance` - On failed consumption
- **Protocol**: REST API + Domain Events via Message Queue
- **Coupling**: Loose (API + events)

**Ownership**:
- Billing Context OWNS financial operations
- Product Contexts OWN product business logic

---

### External Partner: TPayGate (Payment Gateway)

**Relationship Type**: **Partner/External System**

**Interaction Pattern**:
```
[Billing Service] ──(API call)──→ [TPayGate]
                                  ↓
                            [Create Payment]
                                  ↓
[Billing Service] ←─(webhook)─── [TPayGate]
                              ↓
                        [Payment Callback]
```

**Integration Details**:
- **Purpose**: QR code payment processing
- **APIs Called**:
  - `POST {TPayGate}/api/v1/payments` - Create payment
- **Webhook**: `POST /api/v1/webhooks/payment` - Payment callback
- **Security**: HMAC-SHA256 signature verification
- **Retry**: 3 attempts with exponential backoff on API calls
- **SLA**: 99% availability (TPayGate commitment)
- **Protocol**: REST API + Webhooks
- **Coupling**: Synchronous API, asynchronous webhook

**Ownership**:
- TPayGate OWNS payment infrastructure
- Billing Context OWNS payment tracking logic

---

### External Partner: Invoice Hub (Invoice System)

**Relationship Type**: **Partner/External System**

**Interaction Pattern**:
```
[Billing Service] ──(API call)──→ [Invoice Hub]
                                  ↓
                             [Create Invoice]
                                  ↓
[Billing Service] ──(API call)──→ [Invoice Hub]
                                  ↓
                             [Get Invoice Status]
```

**Integration Details**:
- **Purpose**: Electronic invoice generation per Vietnam regulations
- **APIs Called**:
  - `POST {InvoiceHub}/api/v1/invoices` - Create invoice
  - `GET {InvoiceHub}/api/v1/invoices/{id}` - Get invoice status
- **Trigger**: PaymentSucceeded domain event
- **Retry Strategy**: 3 attempts with backoff (30s, 60s, 120s)
- **Fallback**: Manual processing queue
- **SLA**: 99% availability (Invoice Hub commitment)
- **Protocol**: REST API
- **Coupling**: Synchronous API with retry queue

**Ownership**:
- Invoice Hub OWNS invoice infrastructure
- Billing Context OWNS invoice mapping logic

---

## Context Map Diagram

```
┌─────────────────────────────────────────────────────────┐
│                  WION Platform                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐                                     │
│  │ Tenant Service│                                     │
│  │  (Upstream)   │                                     │
│  └───────┬───────┘                                     │
│         │ publishes                                     │
│         ↓                                             │
│  ┌──────────────────┐                               │
│  │ Billing Service │ (Financial Operations)          │
│  │  ├─ Wallet      │                               │
│  │  ├─ Payment     │                               │
│  │  ├─ Credit      │                               │
│  │  ├─ Ledger      │                               │
│  │  └─ Invoice     │                               │
│  └────────┬─────────┘                               │
│           │                                         │
│           │ APIs + Events                          │
│           ↓                                         │
│  ┌──────────────────────────────────────┐       │
│  │       WION Products (Customers)        │       │
│  │  ├─ WION POS                           │       │
│  │  ├─ WION FnB                           │       │
│  │  ├─ WION SPA                           │       │
│  │  ├─ WIPIX                             │       │
│  │  ├─ AI Services                        │       │
│  │  └─ Platform Services                   │       │
│  └──────────────────────────────────────┘       │
│                                                     │
│  ┌────────────────┐  ┌──────────────┐        │
│  │  TPayGate       │  │ Invoice Hub  │        │
│  │  (Partner)      │  │ (Partner)     │        │
│  └────────────────┘  └──────────────┘        │
│         ↕ APIs/Webhooks        ↕ APIs           │
└─────────────────────────────────────────────┘
```

## Communication Patterns

### Pattern 1: Upstream/Downstream (Tenant Service)
- **Direction**: Tenant Service → Billing Service
- **Protocol**: Domain Events
- **Frequency**: On tenant lifecycle events
- **Guarantee**: At-least-once delivery
- **Example**: `TenantCreated` event triggers wallet creation

### Pattern 2: Customer/Supplier (WION Products)
- **Direction**: Billing Service ↔ WION Products
- **Protocol**: REST APIs + Domain Events
- **Frequency**: High (credit operations)
- **Guarantee**: At-least-once delivery for events
- **Example**: Products call `POST /api/v1/credit/consume`, receive `CreditConsumed` event

### Pattern 3: Partner/External (TPayGate)
- **Direction**: Billing Service → TPayGate → Billing Service
- **Protocol**: REST API + Webhooks
- **Frequency**: Medium (payment operations)
- **Guarantee**: Best-effort (retries with backoff)
- **Example**: Create payment API call, webhook callback

### Pattern 4: Partner/External (Invoice Hub)
- **Direction**: Billing Service → Invoice Hub
- **Protocol**: REST API
- **Frequency**: Low (after successful payments)
- **Guarantee**: Best-effort (retry queue)
- **Example`: Create invoice API call on payment success

## Context Ownership Matrix

| Context | Owner | Responsibility | Integration Type |
|---------|--------|---------------|----------------|
| Tenant Service | Platform Team | Tenant lifecycle | Upstream/Downstream |
| Billing Service | Financial Ops Team | Financial operations | This Context |
| WION POS | POS Team | Point of Sale logic | Customer/Supplier |
| WION FnB | FnB Team | Food & Beverage logic | Customer/Supplier |
| WION SPA | SPA Team | Service Platform logic | Customer/Supplier |
| WIPIX | Media Team | Media Platform logic | Customer/Supplier |
| AI Services | AI Team | AI Services logic | Customer/Supplier |
| Platform Services | Platform Team | Platform services logic | Customer/Supplier |
| TPayGate | External Partner | Payment infrastructure | Partner/External |
| Invoice Hub | External Partner | Invoice infrastructure | Partner/External |

## Anti-Corruption Layer

### Purpose
Prevent technical details from external partners and upstream contexts from leaking into Billing Service domain.

### Implementation
- **External Service Adapters**: `infrastructure/external-services/`
- **Domain Events**: Internal domain events, not external events
- **Repositories**: Abstract interfaces in Domain, implementations in Infrastructure

### Examples
- ✅ GOOD: `PaymentCreated` domain event (internal concept)
- ❌ BAD: `TPayGatePaymentCreated` event (external concept leaked)

## Related Documents
- [Bounded Context Definition](./architecture/bounded-context.md)
- [Domain Overview](./domains/) - Business domain details
- [External Services](./infrastructure/external-services.md) - Partner integrations
