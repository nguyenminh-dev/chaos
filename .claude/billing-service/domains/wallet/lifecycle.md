# Wallet Lifecycle

This document describes the complete lifecycle of a Wallet Aggregate from creation to deletion.

## Lifecycle States

```
[Not Created]
      ↓
    [Created]
      ↓
    [Active] ←───┐
      │           │
      │           │ (credit/debit operations)
      │           │
      ↓           │
   [Soft Deleted] │ (7-year retention)
      │           │
      │           └───────────────┘
      ↓
  [Permanently Deleted] (after 7 years)
```

## State Definitions

### Not Created
**Description**: Wallet does not exist for tenant

**Entry**: Initial state before tenant registration

**Exit**: Wallet creation on tenant registration

**Operations**: None

**Events**: None

---

### Created
**Description**: Wallet has been created but not yet activated

**Entry**: Wallet record created in database

**Exit**: Wallet becomes `Active` immediately after creation (no separate activation step)

**Duration**: Transitional state (milliseconds)

**Operations**: None

**Events**: Publishes `WalletCreated` event

**Related**: [Wallet Domain Events](./domain-events.md)

---

### Active
**Description**: Wallet is active and can perform operations

**Entry**: Immediately after creation

**Exit**: Soft delete on tenant deletion

**Operations Allowed**:
- `credit(amount)` - Add funds
- `debit(amount)` - Deduct funds
- `reserve(amount)` - Reserve for pending transaction
- `releaseReserve(amount)` - Release reserved funds
- `checkBalance(amount)` - Validate sufficient balance
- `getTotalBalance()` - Query total balance

**Events Published**:
- `BalanceChanged` - On credit/debit operations
- `InsufficientBalance` - On failed debit due to insufficient funds

**Duration**: From creation until tenant deletion

**Constraints**:
- Balance must be non-negative: [BR-W-002](./business-rules.md)
- Reserved balance must be non-negative: [BR-W-003](./business-rules.md)

**Related**: [Wallet Business Rules](./business-rules.md)

---

### Soft Deleted
**Description**: Wallet is marked as deleted but retained for audit

**Entry**: Tenant deletion triggers wallet soft delete

**Exit**: Permanent deletion after 7 years

**Operations Allowed**:
- Read-only queries (for audit)
- No modifications allowed

**Events Published**: None (soft delete is internal)

**Duration**: 7 years (compliance requirement)

**Database State**: `isDeleted = true`

**Purpose**: Financial compliance and audit trail

**Related**: [BR-W-010](./business-rules.md) - Soft Delete Retention

---

### Permanently Deleted
**Description**: Wallet is permanently removed from database

**Entry**: After 7-year retention period expires

**Exit**: Final state (no transitions)

**Operations**: None

**Events**: None

**Purpose**: Data cleanup after compliance retention period

---

## Lifecycle Operations

### Creation
**Trigger**: Tenant registration event

**Process**:
1. Validate tenant does not already have a wallet
2. Create wallet with zero balance
3. Initialize default assets (WI_CREDIT)
4. Publish `WalletCreated` event
5. Create initial ledger entry

**Preconditions**:
- Tenant must exist
- No wallet exists for tenant

**Postconditions**:
- Wallet record created with `balance = 0`, `reservedBalance = 0`
- Wallet is in `Active` state
- `WalletCreated` event published

**Errors**:
- If wallet already exists: Return error "Wallet already exists"

---

### Deletion
**Trigger**: Tenant deletion event

**Process**:
1. Load wallet by tenant ID
2. Validate wallet can be deleted (no pending operations)
3. Set `isDeleted = true`
4. Soft delete all associated assets
5. Stop all operations on wallet

**Preconditions**:
- No pending transactions (reservedBalance = 0)
- No active operations in progress

**Postconditions**:
- Wallet marked as deleted
- All operations rejected
- Data retained for 7 years

**Errors**:
- If pending transactions exist: Reject deletion

---

## State Transitions

### Valid Transitions
| From State | To State | Trigger |
|------------|----------|---------|
| Not Created | Created | Tenant registration |
| Created | Active | Immediate (automatic) |
| Active | Soft Deleted | Tenant deletion |
| Soft Deleted | Permanently Deleted | 7-year retention expires |

### Invalid Transitions
| Attempted Transition | Reason |
|----------------------|---------|
| Soft Deleted → Active | Cannot restore soft-deleted wallet |
| Permanently Deleted → Any State | Cannot restore permanently deleted wallet |
| Any → Not Created | Cannot return to non-existent state |

---

## Balance Lifecycle

### Available Balance State
```
[Initial: 0]
      ↓
  [Credit] ──────→ [Balance Increased]
      ↓
  [Debit] ──────→ [Balance Decreased]
      ↓
  [Reserve] ────→ [Available ↓, Reserved ↑]
      ↓
  [Release] ────→ [Available ↑, Reserved ↓]
```

**Balance Operations**:
- `credit(amount)` - Increases available balance
- `debit(amount)` - Decreases available balance
- `reserve(amount)` - Transfers from available to reserved
- `releaseReserve(amount)` - Transfers from reserved to available

---

## Asset Lifecycle

### Asset Creation
**Trigger**: First time specific asset type is added to wallet

**Process**:
1. Check if asset of type already exists
2. Create WalletAsset with initial balance
3. Set expiration if applicable

**Asset Type Lifecycle**:
```
[Asset Created]
      ↓
   [Active]
      ↓
[Expired?] → Yes → [Cannot be consumed]
      ↓ No
  [Continue Usage]
```

---

## Lifecycle Events

See [Wallet Domain Events](./domain-events.md) for events published during lifecycle transitions:
- `WalletCreated` - On wallet creation
- `BalanceChanged` - On balance modification
- `InsufficientBalance` - On failed balance validation

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition
- [Wallet Business Rules](./business-rules.md) - Business invariants
- [Wallet Model](./model.md) - Entity and Value Object definitions
- [Wallet Domain Events](./domain-events.md) - Published events
