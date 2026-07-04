# WION Billing Platform - Knowledge Base

## Overview
This is the **comprehensive knowledge base** for the WION Billing Platform - a unified financial platform serving the entire WION ecosystem.

## What This Documentation Covers

### 📋 For AI Agents (Claude Code)
This knowledge base is optimized for AI consumption to enable:
- Understanding business context and architecture
- Performing impact analysis
- Generating implementation plans
- Reviewing code changes
- Updating documentation automatically

### 🏗️ Platform Scope
Billing Platform serves these WION products:
- **WION POS** - Point of Sale
- **WION FnB** - Food & Beverage
- **WION SPA** - Service Platform
- **WIPIX** - Media Platform
- **AI Services** - AI Generation, OCR
- **Platform Services** - SMS, Storage, Marketplace

---

## Documentation Structure

```
chaos/
├── README.md                          # This file
├── billing-service/                    # Main service documentation
│   ├── service.md                     # Service overview
│   ├── api-index.md                   # API catalog
│   ├── event-index.md                 # Event catalog
│   ├── dependency.md                  # Dependencies & integrations
│   ├── database.md                    # Database schema
│   ├── feature-index.md               # Feature catalog
│   ├── glossary.md                    # Business & technical terms
│   ├── architecture.md                # System architecture
│   └── features/                      # Detailed feature docs
│       ├── wallet-management/         # Wallet operations
│       ├── credit-consumption/        # Credit operations
│       ├── payment-processing/         # Payment handling
│       ├── webhook-handling/          # Webhook processing
│       └── invoice-generation/       # Invoice operations
└── .claude/
    ├── user-story/                    # Original requirements
    └── architecture/                 # Architecture diagrams
```

---

## Quick Start for AI Agents

### Understanding the Platform
1. Start with [Service Overview](billing-service/service.md)
2. Review [Feature Index](billing-service/feature-index.md)
3. Check [Glossary](billing-service/glossary.md) for terms

### Implementing Features
1. Read [Feature Overview](billing-service/features/*/overview.md)
2. Review [API Documentation](billing-service/features/*/api.md)
3. Check [Business Rules](billing-service/features/*/business-rule.md)
4. Follow [Workflow](billing-service/features/*/workflow.md)

### Making Changes
1. Perform impact analysis using [Dependency Documentation](billing-service/dependency.md)
2. Check affected APIs in [API Index](billing-service/api-index.md)
3. Review events in [Event Index](billing-service/event-index.md)
4. Update relevant feature documentation

---

## Business Context

### Purpose
Billing Platform is the **central financial hub** for WION, handling:
- 💳 **Payments**: QR code topup via TPayGate
- 💰 **Wallets**: Digital wallet and balance management
- ⚡ **Credits**: Real-time credit consumption (10,000 TPS)
- 📄 **Invoices**: Electronic invoice generation
- 📊 **Ledger**: Double-entry accounting audit trail

### Key Principles
1. **Balance Never Negative**: BR-001 enforced at database level
2. **Double-Entry Accounting**: Every transaction creates ledger entries
3. **Idempotency**: All mutations support idempotency keys
4. **Invoice on Success**: Invoices only created for successful payments

### External Integrations
- **TPayGate**: Payment gateway (QR payments)
- **Invoice Hub**: Electronic invoice system

---

## Technology Stack

### Core Technology
- **Language**: Node.js/TypeScript (Phase 1)
- **Database**: PostgreSQL with read replicas
- **Cache**: Redis (60s TTL for balance)
- **Message Queue**: RabbitMQ/Kafka
- **External**: TPayGate, Invoice Hub

### Performance Targets
- Wallet API: < 100ms (P95)
- Credit API: < 100ms (P95)
- Payment API: < 500ms (P95)
- Webhook: < 1s (P95)
- Throughput: 10,000 consume TPS

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Billing Platform                    │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Wallet  │  │ Payment  │  │  Credit  │        │
│  │  Module  │  │  Module  │  │  Module  │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│       │              │              │                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Webhook  │  │ Invoice  │  │  Ledger  │        │
│  │  Module  │  │  Module  │  │  Module  │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                       │
├─────────────────────────────────────────────────────┤
│                    Client SDK                        │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   POS    │  │   FnB    │  │    AI    │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                       │
└─────────────────────────────────────────────────────┘
```

---

## Key Features

### 1. Wallet Management
- Automatic wallet creation
- Multi-asset support (Wi Credit, Promotion, Gift, Trial)
- Real-time balance queries with caching

### 2. Credit Consumption
- High-performance consume API (10,000 TPS)
- Refund support
- Admin adjustments
- Double-entry ledger

### 3. Payment Processing
- QR code payments via TPayGate
- Payment tracking
- Automatic wallet crediting

### 4. Webhook Handling
- HMAC-SHA256 signature verification
- Idempotent processing
- Automatic wallet credit on success

### 5. Invoice Generation
- Automatic invoice creation
- Retry mechanism (3 attempts)
- Manual processing queue

---

## API Endpoints Summary

### Wallet APIs
- `GET /api/v1/wallets/{tenantId}/balance` - Get balance
- `GET /api/v1/wallets/{tenantId}` - Get wallet info

### Credit APIs
- `POST /api/v1/credit/consume` - Consume credits
- `POST /api/v1/credit/refund` - Refund credits
- `POST /api/v1/credit/adjust` - Adjust balance (admin)
- `GET /api/v1/credit/consumption` - Consumption history

### Payment APIs
- `POST /api/v1/payments/topup` - Initiate topup
- `GET /api/v1/payments/{paymentId}` - Get payment status
- `GET /api/v1/payments` - Payment history

### Invoice APIs
- `GET /api/v1/invoices/payment/{paymentId}` - Get invoice
- `GET /api/v1/invoices/{invoiceNumber}` - Get by number
- `GET /api/v1/invoices` - Invoice history

**Full API Documentation**: [API Index](billing-service/api-index.md)

---

## Events Summary

### Published Events
- `WalletCreated`, `BalanceChanged`
- `PaymentCreated`, `PaymentSucceeded`, `PaymentFailed`, `PaymentExpired`
- `CreditConsumed`, `CreditRefunded`, `InsufficientBalance`
- `TransactionCreated`, `TransactionCompleted`, `TransactionFailed`
- `InvoiceIssued`, `InvoiceFailed`
- `WebhookReceived`, `WebhookProcessed`, `DuplicateWebhookDetected`

### Subscribed Events
- `TenantDeleted` - Trigger wallet cleanup

**Full Event Documentation**: [Event Index](billing-service/event-index.md)

---

## Business Rules Summary

| Rule ID | Description |
|---------|-------------|
| BR-001 | Balance never negative |
| BR-002 | All operations create ledger entries |
| BR-003 | Invoice only on payment success |
| BR-004 | Webhook must be idempotent |
| BR-005 | Reference ID uniqueness |

---

## Development Roadmap

### Completed (Specification Phase)
- ✅ Complete service specification
- ✅ API definitions
- ✅ Database schema
- ✅ Business rules
- ✅ Event contracts

### Sprint 1 (Week 1-2) - Foundation
- ⏳ Wallet Module implementation
- ⏳ Ledger Module implementation
- ⏳ Database setup

### Sprint 2 (Week 3-4) - Core
- ⏳ Payment Module implementation
- ⏳ Credit Module implementation
- ⏳ TPayGate integration

### Sprint 3 (Week 5-6) - Integration
- ⏳ Webhook Module implementation
- ⏳ Invoice Module implementation
- ⏳ Invoice Hub integration

### Sprint 4 (Week 7-8) - Developer Experience
- ⏳ TypeScript SDK implementation
- ⏳ Integration testing
- ⏳ Documentation completion

---

## Support & Contact

- **Platform Team**: platform@wion.vn
- **Tech Lead**: tech-lead@wion.vn
- **Product Owner**: po@wion.vn

---

## License & Copyright

Copyright © 2026 WION Platform Team. All rights reserved.

---

*Last Updated: 2026-07-04*
