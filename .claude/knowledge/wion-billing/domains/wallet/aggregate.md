# Wallet Aggregate

## Purpose
Manage user wallet and balances with support for multiple asset types using Domain-Driven Design principles.

## Aggregate Root
**`Wallet`** - The root entity that provides access to the Wallet Aggregate

**Base Classes**: 
- `BillingAggregateRoot` (extends `TMTFullAuditedAggregateRoot<long>`) - Provides audit trail and soft delete infrastructure

## Entities

### Wallet (Aggregate Root)
**Purpose**: User wallet with balance tracking

**Responsibilities**:
- Maintain available balance
- Maintain reserved balance for pending transactions
- Enforce balance invariants through business rules
- Manage currency
- Publish domain events for state changes
- Manage collection of wallet assets

**Key Operations**:

#### Public Methods
- `CreateNew(string userId, string currency = "VND")` - Static factory method to create new wallet
- `Credit(decimal amount)` - Add funds to available balance
- `Reserve(decimal amount)` - Reserve funds for pending transaction
- `ConfirmDeduction(decimal amount)` - Confirm deduction after service completion
- `ReleaseReserve(decimal amount)` - Release reserved funds on failure
- `AddAsset(AssetType assetType, decimal initialBalance, DateTime? expiresAt)` - Add new asset type to wallet
- `SoftDelete()` - Soft delete wallet
- `HasSufficientBalance(decimal amount)` - Check if wallet has sufficient balance

#### Internal Methods (used by domain logic)
- `HasAsset(AssetType assetType)` - Check if wallet has specific asset type
- `GetAsset(AssetType assetType)` - Get asset of specific type

**Domain Events Published**:
- `WalletCreatedDomainEvent` - When new wallet created
- `BalanceChangedDomainEvent` - When balance changes (credit/reserve/confirm/release)
- `WalletDeletedDomainEvent` - When wallet is soft deleted

---

### WalletAsset
**Purpose**: Multi-asset support within a wallet

**Responsibilities**:
- Track balance per asset type
- Support different asset lifecycles
- Manage asset expiration (for promotional/trial credits)
- Maintain asset-specific reserved balances

**Asset Types Supported**:
- `WI_CREDIT` - Standard WION credits (never expires)
- `PROMOTION` - Promotional credits with expiration
- `GIFT` - Gift credits from campaigns (can expire)
- `TRIAL` - Trial credits for new users (can expire)
- `AI_TOKEN` - AI service tokens (never expires)

**Internal Operations**:
- `Reserve(decimal amount)` - Reserve asset funds
- `ConfirmDeduction(decimal amount)` - Confirm deduction
- `ReleaseReserve(decimal amount)` - Release reserved funds
- `Credit(decimal amount)` - Add funds to asset
- `IsValidForExpirationCheck()` - Check if asset should be evaluated for expiration

## Value Objects

### AssetType
**Purpose**: Asset classification Value Object

**Properties**:
- `Code` - Unique asset type code
- `Name` - Human-readable name
- `CanExpire` - Whether asset type supports expiration

**Asset Type Catalog**:
| Code | Name | Can Expire | Description |
|-----|-----|------------|-------------|
| WI_CREDIT | Wi Credit | No | Standard WION internal payment unit (1 VNĐ = 1 Wi Credit) |
| PROMOTION | Promotion | Yes | Promotional credits with expiration |
| GIFT | Gift | Yes | Gift credits from campaigns |
| TRIAL | Trial | Yes | Trial credits for new users |
| AI_TOKEN | AI Token | No | AI service tokens (future) |

## Business Invariants

See [Wallet Business Rules](./business-rules.md) for complete business rules:

### Core Invariants
- `AvailableBalance >= 0` - Available balance cannot be negative
- `ReservedBalance >= 0` - Reserved balance cannot be negative  
- `TotalBalance = AvailableBalance + ReservedBalance` - Total balance calculation
- One wallet per user (enforced by unique constraint)
- One asset per type per wallet (enforced by unique constraint)

### Operational Rules
- Cannot credit negative amounts
- Cannot reserve more than available balance
- Cannot confirm deduction more than reserved balance
- Cannot operate on deleted wallet
- Cannot add duplicate asset types to wallet

### Balance Operations
- **Reserve**: `AvailableBalance -= amount`, `ReservedBalance += amount`
- **Confirm**: `ReservedBalance -= amount`
- **Release**: `ReservedBalance -= amount`, `AvailableBalance += amount`
- **Credit**: `AvailableBalance += amount`

## Lifecycle

See [Wallet Lifecycle](./lifecycle.md) for complete lifecycle management:

**States**:
- **Active**: Wallet is operational (not deleted)
- **Deleted**: Wallet is soft deleted (retained for audit)

**Transitions**:
- Created when user registers
- Updated on credit/reserve/confirm/release operations
- Soft deleted on user deletion (7-year retention for audit)

**Asset Lifecycle**:
- Created when `AddAsset()` is called
- Expires when `ExpiresAt < DateTime.UtcNow` (if applicable)
- Managed by parent wallet lifecycle

## Domain Events

See [Wallet Domain Events](./domain-events.md) for complete event definitions:

**Wallet Lifecycle Events**:
- `WalletCreatedDomainEvent(Guid walletId, string userId)` - New wallet created
- `WalletDeletedDomainEvent(Guid walletId, string userId)` - Wallet soft deleted

**Balance Operation Events**:
- `BalanceChangedDomainEvent(Guid walletId, string userId, decimal amount, BalanceOperation operation, decimal availableBalance, decimal reservedBalance)` - Balance changed

**Event Types** (`BalanceOperation` enum):
- `Credit` - Funds added to available balance
- `Reserve` - Funds reserved for pending transaction
- `ConfirmDeduction` - Reserved funds confirmed as spent
- `ReleaseReserve` - Reserved funds released back to available

## Repositories

See [Wallet Repositories](./repositories.md) for repository interfaces:

**IWalletRepository** - Main repository for Wallet aggregate
- Inherits `IRepository<Wallet>` from ABP Framework
- Provides domain-specific query methods:
  - `FindByUserIdAsync(string userId)` - Find wallet by user
  - `FindByIdAsync(Guid walletId)` - Find wallet by ID
  - `ExistsByUserIdAsync(string userId)` - Check wallet existence
  - `GetActiveWalletsAsync()` - Get all active wallets
  - `GetDeletedWalletsAsync()` - Get all deleted wallets

**No IWalletAssetRepository** - Wallet assets are owned entities, accessed through Wallet aggregate

## Specifications

### Business Rules (Domain Rules)
The Wallet aggregate enforces business rules through rule objects:

- `AssetAlreadyExistsRule` - Validates asset type uniqueness
- `AssetBalanceRule` - Validates sufficient asset balance
- `AssetReservedBalanceRule` - Validates sufficient asset reserved balance
- `CreditAmountMustBePositiveRule` - Validates credit amount is positive
- `SufficientBalanceRule` - Validates sufficient available balance
- `SufficientReservedBalanceRule` - Validates sufficient reserved balance
- `WalletNotDeletedRule` - Validates wallet is not deleted
- `BalanceMustNotBeNegativeRule` - Validates balance non-negative

## Transaction Boundary

**Single wallet per transaction**

All operations on a Wallet Aggregate must be performed within a single database transaction to maintain consistency.

### ABP Framework Unit of Work

The ABP Framework automatically manages transactions around application service methods via the `[UnitOfWork]` attribute:

```csharp
[UnitOfWork]
public async Task CreditBalanceAsync(string userId, decimal amount)
{
    var wallet = await _walletRepository.FindByUserIdAsync(userId);
    wallet.Credit(amount);
    await _walletRepository.UpdateAsync(wallet);
    // Transaction automatically commits on successful completion
}
```

### Transaction Rules

**DO**:
- Perform all operations on single wallet in one transaction
- Use domain events for cross-aggregate operations (eventual consistency)
- Let ABP Framework manage transaction boundaries

**DON'T**:
- Modify multiple wallets in single transaction
- Modify multiple aggregates in single transaction
- Manually manage database transactions

### Consistency Boundaries

**Strong Consistency** (within Wallet aggregate):
- Wallet and its assets
- Balance calculations
- Business rule enforcement

**Eventual Consistency** (across aggregates):
- Cross-wallet operations via domain events
- Integration with other bounded contexts
- External system notifications

## Related Documents
- [Wallet Model](./model.md) - Entities and Value Objects detailed
- [Wallet Business Rules](./business-rules.md) - Business invariants
- [Wallet Lifecycle](./lifecycle.md) - State management
- [Wallet Domain Events](./domain-events.md) - Published events
- [Wallet Repositories](./repositories.md) - Repository interfaces
- [Wallet Overview](./overview.md) - Domain context and scope