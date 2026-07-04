# Wallet Business Rules

This document defines all business rules (invariants) for the Wallet Aggregate. These rules are enforced within the Wallet domain to maintain data integrity and business compliance.

## Core Business Rules

### BR-W-001: One Wallet Per Tenant
**Rule**: Each tenant must have exactly one wallet.

**Enforcement**:
- Wallet creation checks if wallet already exists for tenant
- Only one wallet can be active per tenant at any time

**Violation Handling**:
- Reject duplicate wallet creation with error: "Wallet already exists for tenant"

**Related**: Wallet lifecycle management

---

### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.

**Formal Definition**: `availableBalance >= 0`

**Enforcement Points**:
- `debit(amount)` operation validates: `availableBalance - amount >= 0`
- `reserve(amount)` operation validates: `availableBalance - amount >= 0`

**Violation Handling**:
- Reject operation with error: "Insufficient balance"
- Publish `InsufficientBalance` domain event

**Related**: [Wallet Model](./model.md) - Balance operations

---

### BR-W-003: Reserved Balance Cannot Be Negative
**Rule**: Wallet reserved balance must never be negative.

**Formal Definition**: `reservedBalance >= 0`

**Enforcement Points**:
- `reserve(amount)` operation adds to reserved balance
- `releaseReserve(amount)` operation validates sufficient reserved funds

**Violation Handling**:
- Database constraint enforces at persistence layer
- Application logic prevents negative reservations

**Related**: Wallet reserve/release operations

---

### BR-W-004: One Asset Per Type Per Wallet
**Rule**: Each wallet can have only one asset of each type.

**Formal Definition**: `UNIQUE(tenantId, assetType)`

**Enforcement Points**:
- Database unique constraint on `(tenant_id, asset_type)`
- Asset creation checks for existing asset of same type

**Violation Handling**:
- Database rejects duplicate asset creation
- Application returns error: "Asset of type X already exists"

**Related**: [Wallet Model](./model.md) - WalletAsset entity

---

## Derived Business Rules

### BR-W-005: Total Balance Calculation
**Rule**: Total wallet balance equals available balance plus reserved balance.

**Formal Definition**: `totalBalance = availableBalance + reservedBalance`

**Purpose**: Maintain consistency between balance components

**Enforcement**: Computed value, not stored

---

### BR-W-006: Asset-Specific Balance Non-Negative
**Rule**: Individual asset balances cannot be negative.

**Formal Definition**: For each WalletAsset: `balance >= 0` and `reservedBalance >= 0`

**Enforcement Points**:
- Database constraints on `wallet_asset` table
- Asset-level operations validate before execution

**Violation Handling**:
- Database constraint violation
- Application-level validation before persistence

**Related**: [Wallet Model](./model.md) - WalletAsset entity

---

### BR-W-007: Asset Expiration
**Rule**: Expired assets cannot be consumed.

**Formal Definition**: `IF expiresAt IS NOT NULL AND NOW() > expiresAt THEN asset.isExpired = true`

**Enforcement Points**:
- Asset balance queries filter out expired assets
- Consumption operations check asset expiration status

**Violation Handling**:
- Reject consumption from expired assets
- Return error: "Asset has expired"

**Asset Types with Expiration**:
- `PROMOTION` - Can expire
- `GIFT` - Can expire
- `TRIAL` - Can expire
- `WI_CREDIT` - Never expires
- `AI_TOKEN` - Never expires

---

## Cross-Asset Business Rules

### BR-W-008: Debit Priority Order
**Rule**: When debiting wallet, consume assets in priority order.

**Priority Order**:
1. `TRIAL` - Trial credits first
2. `PROMOTION` - Promotional credits second
3. `GIFT` - Gift credits third
4. `WI_CREDIT` - Standard credits last
5. `AI_TOKEN` - AI tokens last

**Purpose**: Maximize usage of expiring/limited assets before permanent credits

**Enforcement**: Application-level logic in debit operations

---

## Operational Business Rules

### BR-W-009: Reserve Release on Failure
**Rule**: Reserved funds must be released back to available balance on service failure.

**Trigger**: Service operation failure after reservation

**Action**: `reservedBalance -= amount; availableBalance += amount`

**Purpose**: Prevent fund lockup on failed operations

**Enforcement**: Application-level compensation logic

---

### BR-W-010: Soft Delete Retention
**Rule**: Deleted wallets must be retained for 7 years for audit purposes.

**Purpose**: Financial compliance and audit trail

**Enforcement**:
- Wallet is soft deleted (`isDeleted = true`)
- Data retained for 7 years before permanent deletion
- All wallet operations reject soft-deleted wallets

---

## Business Rule Format

All business rules follow this format:

```
### BR-W-XXX: [Rule Name]
**Rule**: [Human-readable rule description]

**Formal Definition**: [Mathematical or logical definition]

**Enforcement Points**: [Where/how this rule is enforced]

**Violation Handling**: [What happens when rule is violated]

**Purpose**: [Why this rule exists]

**Enforcement**: [How this rule is implemented]
```

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition
- [Wallet Model](./model.md) - Entity and Value Object definitions
- [Wallet Lifecycle](./lifecycle.md) - State transitions
- [Wallet Domain Events](./domain-events.md) - Events published on rule violations
