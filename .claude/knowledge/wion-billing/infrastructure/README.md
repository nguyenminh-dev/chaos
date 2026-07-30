# Infrastructure Layer

This directory documents the infrastructure layer of the Billing Service, including database schemas, external service integrations, and implementation details.

## Contents

### [Database Schema](./database.md)
PostgreSQL database schema including tables, indexes, and relationships for:
- Wallet and WalletAsset
- Payment and PaymentWebhook
- CreditTransaction
- LedgerEntry
- InvoiceReference

### External Services Integration
- **TPayGate**: Payment gateway integration for QR code payments
- **Invoice Hub**: Electronic invoice system integration

### Caching Strategy
- Redis caching for wallet balances
- Cache invalidation policies
- Performance optimization

### Message Queue
- RabbitMQ event publishing
- Event delivery guarantees
- Retry mechanisms

## Purpose
The infrastructure layer implements the interfaces defined by the domain layer and handles all external concerns including persistence, caching, and third-party integrations.

## Key Principles
- Domain layer has NO dependencies on infrastructure
- Infrastructure implements Domain interfaces
- All external service calls are abstracted behind interfaces
- Database is a detail, not the center
