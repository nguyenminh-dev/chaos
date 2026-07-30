# T-PayGate Integration Domain

## Purpose
Integration with T-PayGate payment gateway provider for QR-based bank payments. T-PayGate acts as an intermediary gateway that abstracts multiple banking partners, providing a single integration point instead of direct bank integrations.

## Implementation Status

🔄 **PLANNED** - Integration specification analyzed, implementation pending

## Vendor Documentation
- **Source**: [T-PayGate Integration Specification](../../../../../stories/billing-service/t-paygate-tai-lieu-tich-hop-cho-doi-tac-thu-3.md)
- **Version**: 1.3 (2026-05-05)
- **Provider**: T-PayGate (TPos)
- **Integration Type**: Payment gateway provider/aggregator service

## Domain Scope

### What This Domain Handles
- OAuth token management with T-PayGate gateway
- Bank connection establishment and lifecycle management (via T-PayGate)
- Payment bill (QR code) creation and management (via T-PayGate)
- Webhook processing for payment notifications from T-PayGate
- Bank configuration and credential management (stored for T-PayGate)
- Integration resilience and error handling with T-PayGate

### What This Domain Does NOT Handle
- Customer-facing payment UI (handled by external systems)
- Direct bank communication (handled by T-PayGate gateway)
- Actual payment processing (handled by banking partners via T-PayGate)
- Payment settlement/reconciliation (handled by Ledger domain)
- Invoice generation (handled by Invoice domain)
- Credit operations (handled by Credit domain)
- Bank selection/routing (handled by T-PayGate gateway)

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              T-PayGate Payment Gateway Integration Architecture               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  [Billing Service]              [T-PayGate Gateway]     [Banking Partners]  │
│       │                                 │                      │             │
│  ┌────▼────┐                        ┌──────▼──────┐          │             │
│  │  OAuth   │                        │   OAuth     │          │             │
│  │  Service │◄───────────────────────│   Service   │          │             │
│  └────┬────┘                        └──────┬──────┘          │             │
│       │                                      │                      │             │
│  ┌────▼────┐                        ┌──────▼──────┐          │             │
│  │  Bank    │                        │  Bank Conn  │─────────►│             │
│  │Connect   │◄───────────────────────│   Service   │          │             │
│  └────┬────┘                        └──────┬──────┘          │             │
│       │                                      │                      │             │
│  ┌────▼────┐                        ┌──────▼──────┐          │             │
│  │  Bill    │                        │   Bill      │─────────►│             │
│  │ Service  │◄───────────────────────│   Service   │          │             │
│  └────┬────┘                        └──────┬──────┘          │             │
│       │                                      │                      │             │
│  ┌────▼────┘                                       │              │             │
│  │ Webhook  │                                      │              │             │
│  │ Handler  │◄───────────────────────────────────────┘              │             │
│  └────┬────┘                                                     │             │
│       │                                                           │             │
│       └────────────────────> Payment Processing ─────────────────┴─────────────┘
│                                                                            │
│  Legend:                                                                   │
│  ━━━► Direct API call                                                       │
│  ──► Bank API call (abstracted by T-PayGate)                                │
│  ──► Payment processing (routed by T-PayGate to banks)                     │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### 1. OAuth Token Lifecycle
- **Authentication Method**: OAuth 2.0 Client Credentials
- **Token TTL**: 3600 seconds (60 minutes)
- **Refresh Strategy**: Proactive refresh at T-5 minutes
- **Scope**: All API calls require valid Bearer token

### 2. Bank Connection Model
- **One-Time Setup**: Per merchant-bank pair
- **Output**: `configBankId` and `vaNumber` (Virtual Account)
- **Lifecycle**: INITIATED → PENDING_OTP → CONNECTED → DISCONNECTED
- **Storage**: Persisted for reuse across all transactions

### 3. Bill/Payment Flow
- **Creation**: Generate QR code for customer payment
- **Idempotency**: `refTransactionId` (client-side, 24h cache)
- **Lifecycle**: CREATED → WAITING_PAYMENT → PAID/EXPIRED/CANCELED
- **Notification**: Webhook callback on payment completion

### 4. Webhook Processing
- **Pattern**: Fire-and-forget with immediate HTTP 200
- **Processing**: Background job with idempotency
- **Retry**: 3 attempts × 5-minute intervals (T-PayGate side)
- **Fallback**: Polling for missed webhooks

## Domain Boundaries

### Aggregates
- **BankConnection**: Manages bank connection lifecycle and credentials
- **Bill**: Handles payment bill creation and status tracking  
- **PaymentNotification**: Manages webhook processing and idempotency

### Value Objects
- `TpgConfigBankId` - Bank connection identifier
- `TpgBillCode` - Bill identifier
- `TpgVaNumber` - Virtual Account number
- `TpgRefTransactionId` - Merchant transaction reference

### Repositories
- `ITpgBankConnectionRepository` - Bank connection persistence
- `ITpgBillRepository` - Bill persistence and queries
- `ITpgPaymentNotificationRepository` - Webhook idempotency

## Related Documents

### Domain Documentation
- [T-PayGate Aggregates](./aggregates.md) - Aggregate definitions and lifecycle
- [T-PayGate Business Rules](./business-rules.md) - Business rules and constraints
- [T-PayGate Domain Events](./domain-events.md) - Domain events published/consumed
- [T-PayGate Domain Model](./model.md) - Detailed domain model

### Integration Documentation
- [T-PayGate API Documentation](../api/external/tpaygate.md) - Complete API reference
- [T-PayGate Integration Flows](../../flows/tpaygate-integration.md) - End-to-end flows
- [T-PayGate Resiliency Patterns](../../infrastructure/tpaygate-resiliency.md) - Circuit breakers, retries
- [T-PayGate Security](../../infrastructure/tpaygate-security.md) - Security considerations

### Reference Documentation
- [T-PayGate Implementation Checklist](../../reference/tpaygate-implementation-checklist.md) - Engineering checklist
- [T-PayGate Testing Scenarios](../../reference/tpaygate-testing.md) - Test scenarios and coverage

### Policies
- [T-PayGate Integration Policies](../../policies/tpaygate-integration.md) - Integration policies and standards

## Integration Points

### Upstream Dependencies
- **T-PayGate Gateway API**: OAuth, bank connection management, bill creation APIs
- **Banking Partners**: Actual payment processing (abstracted by T-PayGate gateway)

### Downstream Consumers
- **Ledger Domain**: Payment transaction recording
- **Invoice Domain**: Invoice generation on successful payment
- **Wallet Domain**: Balance updates (if applicable)

### Payment Flow Architecture
```
Customer → Banking App → [Bank] → [T-PayGate Gateway] → [Billing Service]
                           ↑                ↑                      ↑
                      Payment      Gateway Aggregation      Notification
                      Processing   & Abstraction            Processing
```

**Key Clarification**: T-PayGate does NOT process payments directly. Banking partners are the actual payment providers. T-PayGate provides:
- Single API abstraction for multiple banks
- Bank connection management  
- QR code generation and routing
- Payment notification aggregation

### Event Publications
- `TpgBankConnectionConnected` - Bank connection established
- `TpgBankConnectionDisconnected` - Bank connection terminated
- `TpgBillCreated` - Payment bill created
- `TpgPaymentReceived` - Payment notification processed
- `TpgPaymentCompleted` - Payment fully processed

### Event Subscriptions
- `TenantDeleted` - Cleanup tenant data

## Environment Configuration

### Staging (UAT)
- **Base URL**: `https://t-paygate.tpos.dev`
- **Webhook Timeout**: 30 seconds
- **Purpose**: Integration testing and validation

### Production
- **Base URL**: `https://t-paygate.tpos.app`  
- **Webhook Timeout**: 10 seconds
- **Purpose**: Live payment processing

### Required Configuration
- `clientId` - OAuth client identifier
- `tenantId` - OAuth tenant identifier
- `source` - Channel identifier
- `webhookUrl` - HTTPS webhook endpoint
- `ipWhitelist` - Outbound IP addresses for PROD

## Success Metrics

### Technical Metrics
- **OAuth Token Success Rate**: > 99.9%
- **Bank Connection Success Rate**: > 95%
- **Bill Creation Success Rate**: > 99%
- **Webhook Processing Latency**: < 5 seconds (P95)
- **Webhook Idempotency Rate**: 100%

### Business Metrics
- **Payment Success Rate**: > 98%
- **Payment Notification Latency**: < 30 seconds (P95)
- **Reconciliation Accuracy**: 100%

## See Also
- [Payment Domain Overview](../payment/overview.md) - Parent payment domain
- [Electronic Invoice Provider Abstraction](../electronic-invoice/provider-abstraction.md) - Similar integration pattern
