# Wallet Aggregate

## Purpose
Manage tenant wallet and balances with support for multiple asset types.

## Aggregate Root
**`Wallet`** - The root entity that provides access to the Wallet Aggregate

## Entities

### Wallet (Aggregate Root)
**Purpose**: Tenant wallet with balance tracking

**Responsibilities**:
- Maintain available balance
- Maintain reserved balance for pending transactions
- Enforce balance invariants
- Manage currency

**Key Operations**:
- `reserve(amount)` - Reserve funds for pending transaction
- `confirmDeduction(amount)` - Confirm deduction after service completion
- `releaseReserve(amount)` - Release reserved funds on failure
- `credit(amount)` - Add funds to wallet
- `checkBalance(amount)` - Validate sufficient balance

---

### WalletAsset
**Purpose**: Multi-asset support within a wallet

**Responsibilities**:
- Track balance per asset type
- Support different asset lifecycles
- Manage asset expiration (for promotional/trial credits)

**Asset Types**:
- `WI_CREDIT` - Standard WION credits
- `PROMOTION` - Promotional credits with expiration
- `GIFT` - Gift credits from campaigns
- `TRIAL` - Trial credits for new users
- `AI_TOKEN` - Future: AI service tokens

## Value Objects

See [Wallet Model](./model.md) for detailed Value Object definitions:
- `Balance` - Immutable balance representation
- `Currency` - Currency type (VND)
- `AssetType` - Asset classification

## Business Invariants

See [Wallet Business Rules](./business-rules.md) for complete business rules:
- `balance >= 0` - Balance cannot be negative
- `reserved_balance >= 0` - Reserved balance cannot be negative
- One wallet per tenant
- One asset per type per wallet

## Lifecycle

See [Wallet Lifecycle](./lifecycle.md) for complete lifecycle management:
- Created when tenant registers
- Updated on credit/debit operations
- Soft deleted on tenant deletion (7-year retention)

## Domain Events

See [Wallet Domain Events](./domain-events.md) for complete event definitions:
- `WalletCreated` - New wallet created for tenant
- `BalanceChanged` - Balance updated (credit/debit)
- `InsufficientBalance` - Balance insufficient for operation

## Repositories

See [Wallet Repositories](./repositories.md) for repository interfaces:
- `IWalletRepository` - Load/save wallets
- `IWalletAssetRepository` - Load/save wallet assets

## Specifications

- `ActiveWalletSpec` - Query non-deleted wallets
- `SufficientBalanceSpec` - Check if balance sufficient for amount

## Transaction Boundary
**Single wallet per transaction**

All operations on a Wallet Aggregate must be performed within a single database transaction to maintain consistency.

## Related Documents
- [Wallet Model](./model.md) - Entities and Value Objects detailed
- [Wallet Business Rules](./business-rules.md) - Business invariants
- [Wallet Lifecycle](./lifecycle.md) - State management
- [Wallet Domain Events](./domain-events.md) - Published events
- [Wallet Repositories](./repositories.md) - Repository interfaces
