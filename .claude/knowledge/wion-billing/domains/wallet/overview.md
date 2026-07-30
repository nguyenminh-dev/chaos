# Wallet Domain

## Purpose
The Wallet domain manages tenant digital wallets and balances for the WION ecosystem. It is responsible for storing, tracking, and updating wallet balances across multiple asset types.

## Scope
The Wallet domain encompasses:
- Wallet creation and lifecycle management
- Balance tracking (available and reserved)
- Multi-asset support (Wi Credit, Promotion, Gift, Trial, AI Token)
- Balance validation and enforcement of business rules

## Bounded Context
Wallet operates within the **Financial Operations** bounded context, handling all wallet-related operations for the Billing Service.

## Ubiquitous Language
- **Wallet**: Digital wallet belonging to a tenant
- **Balance**: Available funds for consumption
- **Reserved Balance**: Funds temporarily locked for pending transactions
- **Asset**: Different types of credits stored in wallet
- **Wi Credit**: Standard WION internal payment unit (1 VNĐ = 1 Wi Credit)

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Complete Aggregate definition
- [Wallet Model](./model.md) - Entities and Value Objects
- [Wallet Business Rules](./business-rules.md) - Business invariants
- [Wallet Lifecycle](./lifecycle.md) - State transitions
- [Wallet Domain Events](./domain-events.md) - Events published
- [Wallet Repositories](./repositories.md) - Repository interfaces
