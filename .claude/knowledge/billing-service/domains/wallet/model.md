# Wallet Model

This document describes the Entities and Value Objects that comprise the Wallet Aggregate.

## Entities

### Wallet (Aggregate Root)

**Purpose**: Tenant wallet with balance tracking

**Attributes**:
- `tenantId: string` - Unique tenant identifier (PRIMARY KEY)
- `availableBalance: decimal` - Available funds for consumption
- `reservedBalance: decimal` - Funds locked for pending transactions
- `currency: string` - Currency type (default: VND)
- `createdAt: datetime` - Wallet creation timestamp
- `updatedAt: datetime` - Last update timestamp
- `version: int` - Optimistic locking version
- `isDeleted: boolean` - Soft delete flag

**Key Operations**:
- `reserve(amount: decimal): void` - Reserve funds for pending transaction
- `confirmDeduction(amount: decimal): void` - Confirm deduction after service completion
- `releaseReserve(amount: decimal): void` - Release reserved funds on failure
- `credit(amount: decimal): void` - Add funds to available balance
- `debit(amount: decimal): void` - Deduct from available balance
- `checkBalance(amount: decimal): boolean` - Validate sufficient balance
- `getTotalBalance(): decimal` - Get total balance (available + reserved)

**Business Rules Enforced**:
- Available balance cannot be negative: `availableBalance >= 0`
- Reserved balance cannot be negative: `reservedBalance >= 0`
- Total balance: `totalBalance = availableBalance + reservedBalance`

---

### WalletAsset

**Purpose**: Multi-asset support within a wallet

**Attributes**:
- `id: string` - Unique asset identifier (PRIMARY KEY)
- `tenantId: string` - Tenant identifier (FOREIGN KEY → wallet.tenantId)
- `assetType: AssetType` - Asset classification
- `balance: decimal` - Asset-specific balance
- `reservedBalance: decimal` - Asset-specific reserved balance
- `metadata: json` - Additional asset properties
- `createdAt: datetime` - Asset creation timestamp
- `updatedAt: datetime` - Last update timestamp
- `expiresAt: datetime?` - Expiration date (nullable)
- `isDeleted: boolean` - Soft delete flag

**Key Operations**:
- `reserve(amount: decimal): void` - Reserve asset funds
- `confirmDeduction(amount: decimal): void` - Confirm deduction
- `releaseReserve(amount: decimal): void` - Release reserve
- `credit(amount: decimal): void` - Add funds
- `isExpired(): boolean` - Check if asset has expired

**Constraints**:
- One asset per type per wallet: `(tenantId, assetType)` must be unique
- Balance cannot be negative: `balance >= 0`
- Reserved balance cannot be negative: `reservedBalance >= 0`

**Lifecycle**:
- Created when wallet receives specific asset type
- Expires if `expiresAt` is set and current time > `expiresAt`
- Soft deleted when wallet is deleted

---

## Value Objects

### Balance

**Purpose**: Immutable balance representation

**Attributes**:
- `amount: decimal` - Balance amount
- `currency: Currency` - Currency type

**Invariant**:
- Amount cannot be negative: `amount >= 0`

**Operations**:
- `add(amount: decimal): Balance` - Create new balance with added amount
- `subtract(amount: decimal): Balance` - Create new balance with subtracted amount
- `isGreaterThan(other: Balance): boolean` - Comparison operation
- `isLessThan(other: Balance): boolean` - Comparison operation

**Immutability**: All operations return new Balance instances

---

### Currency

**Purpose**: Currency type representation

**Valid Values**:
- `VND` - Vietnamese Dong (default)
- Future: USD, EUR, etc.

**Constraints**:
- Currency codes follow ISO 4217 standard

**Operations**:
- `isValid(): boolean` - Validate currency code
- `getSymbol(): string` - Get currency symbol

---

### AssetType

**Purpose**: Asset classification enum

**Valid Values**:
- `WI_CREDIT` - Standard WION credits
- `PROMOTION` - Promotional credits with expiration
- `GIFT` - Gift credits from campaigns
- `TRIAL` - Trial credits for new users
- `AI_TOKEN` - AI service tokens (future)

**Attributes**:
- `value: string` - Asset type value
- `expires: boolean` - Whether this asset type can expire
- `transferable: boolean` - Whether this asset type can be transferred

**Operations**:
- `canExpire(): boolean` - Check if asset type supports expiration
- `isTransferable(): boolean` - Check if asset type can be transferred

**Asset Type Rules**:
- `WI_CREDIT` - Never expires, transferable
- `PROMOTION` - Can expire, not transferable
- `GIFT` - Can expire, transferable
- `TRIAL` - Can expire, not transferable
- `AI_TOKEN` - Never expires, transferable

---

## Relationships

```
Wallet (1) ──── (1..n) WalletAsset
  │                     │
  │                     │
  └── tenantId ──────── tenantId
```

**Relationship Rules**:
- One wallet per tenant (1:1)
- One-to-many: Wallet → WalletAsset
- Cascade delete: When wallet is deleted, all assets are soft deleted

---

## Database Mapping

### Wallet Table
```sql
CREATE TABLE wallet (
  tenant_id VARCHAR(255) PRIMARY KEY,
  balance DECIMAL(19,4) NOT NULL DEFAULT 0,           -- availableBalance
  reserved_balance DECIMAL(19,4) NOT NULL DEFAULT 0,  -- reservedBalance
  currency VARCHAR(3) NOT NULL DEFAULT 'VND',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  version INT NOT NULL DEFAULT 0,
  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

  CONSTRAINT chk_balance_nonnegative CHECK (balance >= 0),
  CONSTRAINT chk_reserved_nonnegative CHECK (reserved_balance >= 0)
);
```

### WalletAsset Table
```sql
CREATE TABLE wallet_asset (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  asset_type ENUM('WI_CREDIT', 'PROMOTION', 'GIFT', 'TRIAL', 'AI_TOKEN') NOT NULL,
  balance DECIMAL(19,4) NOT NULL DEFAULT 0,
  reserved_balance DECIMAL(19,4) NOT NULL DEFAULT 0,
  metadata JSON,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NULL,
  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

  CONSTRAINT fk_wallet FOREIGN KEY (tenant_id) REFERENCES wallet(tenant_id) ON DELETE CASCADE,
  CONSTRAINT uq_tenant_asset UNIQUE (tenant_id, asset_type),
  CONSTRAINT chk_asset_balance_nonnegative CHECK (balance >= 0)
);
```

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition and operations
- [Wallet Business Rules](./business-rules.md) - Business invariants
- [Wallet Domain Events](./domain-events.md) - Events published by Wallet
