# Wallet Business Rules

This document defines all business rules (invariants) for the Wallet Aggregate. These rules are enforced within the Wallet domain to maintain data integrity and business compliance.

## Core Business Rules

### BR-W-001: One Wallet Per User
**Rule**: Each user must have exactly one wallet.

**Formal Definition**: `UNIQUE(UserId)` in Wallets table

**Enforcement Points**:
- Database unique constraint on `UserId` column
- `IWalletRepository.ExistsByUserIdAsync()` checks existence before creation
- Wallet creation should validate user doesn't have existing wallet

**Violation Handling**:
- Database rejects duplicate wallet creation
- Application should return error: "Wallet already exists for user {userId}"

**Related**: 
- [Wallet Model](./model.md) - Wallet entity
- [Wallet Repositories](./repositories.md) - Repository validation methods

---

### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.

**Formal Definition**: `AvailableBalance >= 0`

**Enforcement Points**:
- Enforced by `BalanceMustNotBeNegativeRule` business rule
- Applied in all balance modification operations
- Database constraint: `CONSTRAINT CK_Wallets_AvailableBalance CHECK (AvailableBalance >= 0)`

**Violation Handling**:
- Business rule throws exception before balance becomes negative
- Operation rejected with specific error message
- No database write operation occurs

**Implementation**:
```csharp
// Enforced in operations that could make balance negative
public void Credit(decimal amount)
{
    CheckRule(new CreditAmountMustBePositiveRule(amount));
    CheckRule(new WalletNotDeletedRule(this));
    AvailableBalance += amount;
    // ... publishes BalanceChangedDomainEvent
}
```

**Related**: 
- [Wallet Model](./model.md) - Balance operations
- Business Rules: `CreditAmountMustBePositiveRule`, `BalanceMustNotBeNegativeRule`

---

### BR-W-003: Reserved Balance Cannot Be Negative
**Rule**: Wallet reserved balance must never be negative.

**Formal Definition**: `ReservedBalance >= 0`

**Enforcement Points**:
- Enforced by `SufficientReservedBalanceRule` in deduction/release operations
- Database constraint: `CONSTRAINT CK_Wallets_ReservedBalance CHECK (ReservedBalance >= 0)`

**Violation Handling**:
- Business rule validation before operation execution
- Operation rejected with error: "Insufficient reserved balance"

**Implementation**:
```csharp
public void ConfirmDeduction(decimal amount)
{
    CheckRule(new SufficientReservedBalanceRule(ReservedBalance, amount));
    CheckRule(new WalletNotDeletedRule(this));
    ReservedBalance -= amount;
    // ... publishes BalanceChangedDomainEvent
}
```

**Related**: 
- Business Rules: `SufficientReservedBalanceRule`

---

### BR-W-004: Sufficient Balance Required
**Rule**: Cannot reserve more funds than available in wallet.

**Formal Definition**: `AvailableBalance >= amount` for Reserve operations

**Enforcement Points**:
- Enforced by `SufficientBalanceRule` before reservation
- Prevents overdraft of available balance

**Violation Handling**:
- Business rule throws exception: "Insufficient balance available"
- Reserve operation rejected
- No domain event published

**Implementation**:
```csharp
public void Reserve(decimal amount)
{
    CheckRule(new SufficientBalanceRule(AvailableBalance, amount));
    CheckRule(new WalletNotDeletedRule(this));
    AvailableBalance -= amount;
    ReservedBalance += amount;
    // ... publishes BalanceChangedDomainEvent
}
```

**Related**: 
- Business Rules: `SufficientBalanceRule`

---

### BR-W-005: One Asset Per Type Per Wallet
**Rule**: Each wallet can have only one asset of each type.

**Formal Definition**: `UNIQUE(AssetTypeCode, WalletId)` in WalletAssets table

**Enforcement Points**:
- Database unique constraint: `CONSTRAINT UQ_WalletAssets_Type UNIQUE (AssetTypeCode, WalletId)`
- Enforced by `AssetAlreadyExistsRule` in domain
- Asset creation checks for existing asset of same type

**Violation Handling**:
- Business rule throws exception: "Asset of type {assetType} already exists"
- Asset addition rejected
- Database constraint prevents duplicates at persistence layer

**Implementation**:
```csharp
public void AddAsset(AssetType assetType, decimal initialBalance, DateTime? expiresAt)
{
    CheckRule(new AssetAlreadyExistsRule(Assets, assetType));
    var asset = new WalletAsset(Id, assetType, initialBalance, expiresAt);
    Assets.Add(asset);
    
    if (initialBalance > 0)
    {
        Credit(initialBalance);
    }
}
```

**Related**: 
- [Wallet Model](./model.md) - WalletAsset entity
- Business Rules: `AssetAlreadyExistsRule`

---

### BR-W-006: Credit Amount Must Be Positive
**Rule**: Can only credit positive amounts to wallet balance.

**Formal Definition**: `amount > 0` for all Credit operations

**Enforcement Points**:
- Enforced by `CreditAmountMustBePositiveRule`
- Applied in both `Wallet.Credit()` and `WalletAsset.Credit()`

**Violation Handling**:
- Operation rejected with error: "Credit amount must be positive"
- No balance modification occurs

**Implementation**:
```csharp
public void Credit(decimal amount)
{
    CheckRule(new CreditAmountMustBePositiveRule(amount));
    CheckRule(new WalletNotDeletedRule(this));
    AvailableBalance += amount;
    AddLocalEvent(new BalanceChangedDomainEvent(...));
}
```

**Related**: 
- Business Rules: `CreditAmountMustBePositiveRule`

---

### BR-W-007: Cannot Operate On Deleted Wallet
**Rule**: Deleted wallets cannot be modified.

**Formal Definition**: `IsDeleted == false` for all balance operations

**Technical Implementation**: 
- Uses `IsDeleted` field from `TMTFullAuditedAggregateRoot<long>` base class
- ABP Framework's `IDataFilter` automatically excludes deleted wallets from queries

**Enforcement Points**:
- Enforced by `WalletNotDeletedRule` in all modifying operations
- Checked at domain level before any state change
- ABP Framework prevents querying deleted wallets in normal operations

**Violation Handling**:
- Operation rejected with error: "Cannot operate on deleted wallet"
- No state change occurs
- No domain events published

**Implementation**:
```csharp
public void Credit(decimal amount)
{
    CheckRule(new CreditAmountMustBePositiveRule(amount));
    CheckRule(new WalletNotDeletedRule(this)); // Validates technical field
    AvailableBalance += amount;
    // ...
}
```

**Related**: 
- Business Rules: `WalletNotDeletedRule`
- [Wallet Lifecycle](./lifecycle.md) - Soft delete process

---

## Derived Business Rules

### BR-W-008: Total Balance Calculation
**Rule**: Total wallet balance equals available balance plus reserved balance.

**Formal Definition**: `TotalBalance = AvailableBalance + ReservedBalance`

**Purpose**: Maintain consistency between balance components for reporting

**Enforcement**: Computed property, not stored in database

**Implementation**:
```csharp
public decimal TotalBalance => AvailableBalance + ReservedBalance;
```

**Usage**: Reporting, balance inquiries, validation

---

### BR-W-009: Asset-Specific Balance Non-Negative
**Rule**: Individual asset balances cannot be negative.

**Formal Definition**: For each WalletAsset: `Balance >= 0` and `ReservedBalance >= 0`

**Enforcement Points**:
- Database constraints: `CONSTRAINT CK_WalletAssets_Balance CHECK (Balance >= 0)`
- Enforced by `AssetBalanceRule` and `AssetReservedBalanceRule` in asset operations
- Applied at asset level for reserve/confirm/release operations

**Violation Handling**:
- Business rule validation before asset operations
- Operation rejected with error: "Insufficient asset balance"
- Database constraint prevents persistence of negative values

**Implementation**:
```csharp
// In WalletAsset entity
internal void Reserve(decimal amount)
{
    BusinessRuleHelper.CheckRule(new AssetBalanceRule(this.Balance, amount));
    Balance -= amount;
    ReservedBalance += amount;
}

internal void ConfirmDeduction(decimal amount)
{
    BusinessRuleHelper.CheckRule(new AssetReservedBalanceRule(this.ReservedBalance, amount));
    ReservedBalance -= amount;
}
```

**Related**: 
- [Wallet Model](./model.md) - WalletAsset entity
- Business Rules: `AssetBalanceRule`, `AssetReservedBalanceRule`

---

### BR-W-010: Asset Expiration
**Rule**: Expired assets cannot be consumed for new operations.

**Formal Definition**: `IF ExpiresAt IS NOT NULL AND DateTime.UtcNow > ExpiresAt THEN IsExpired = true`

**Enforcement Points**:
- Computed property: `public bool IsExpired => ExpiresAt.HasValue && DateTime.UtcNow > ExpiresAt.Value`
- Asset type metadata: `AssetType.CanExpire` indicates which assets can expire
- Validation in asset consumption logic

**Asset Types with Expiration**:
- `PROMOTION` - Can expire (CanExpire: true)
- `GIFT` - Can expire (CanExpire: true)
- `TRIAL` - Can expire (CanExpire: true)

**Asset Types Without Expiration**:
- `WI_CREDIT` - Never expires (CanExpire: false)
- `AI_TOKEN` - Never expires (CanExpire: false)

**Implementation**:
```csharp
public class AssetType : ValueObject
{
    public string Code { get; }
    public string Name { get; }
    public bool CanExpire { get; }
    
    public static AssetType WiCredit() => new AssetType("WI_CREDIT", "Wi Credit", canExpire: false);
    public static AssetType Promotion() => new AssetType("PROMOTION", "Promotion", canExpire: true);
    // ... other factory methods
}
```

**Related**: 
- [Wallet Model](./model.md) - AssetType value object
- [Wallet Lifecycle](./lifecycle.md) - Asset expiration handling

---

## Cross-Asset Business Rules

### BR-W-011: Asset Type Validation
**Rule**: Only predefined asset types are allowed.

**Formal Definition**: Asset type codes must match predefined catalog

**Enforcement Points**:
- AssetType.Of() method validates code against known types
- Throws `ArgumentException` for unknown asset types

**Implementation**:
```csharp
public static AssetType Of(string code)
{
    return code switch
    {
        "WI_CREDIT" => WiCredit(),
        "PROMOTION" => Promotion(),
        "GIFT" => Gift(),
        "TRIAL" => Trial(),
        "AI_TOKEN" => AiToken(),
        _ => throw new ArgumentException($"Unknown asset type: {code}")
    };
}
```

**Violation Handling**:
- Exception thrown: "Unknown asset type: {code}"
- Asset creation rejected

---

### BR-W-012: Asset Balance Consistency
**Rule**: Asset-level operations must maintain balance consistency.

**Formal Definition**: Asset balance operations are internal to Wallet aggregate

**Enforcement Points**:
- WalletAsset operations are internal (not public)
- Wallet aggregate controls all asset operations
- Asset operations called within Wallet methods

**Implementation**:
```csharp
// WalletAsset has internal methods only
internal void Reserve(decimal amount) { /* ... */ }
internal void ConfirmDeduction(decimal amount) { /* ... */ }
internal void ReleaseReserve(decimal amount) { /* ... */ }
internal void Credit(decimal amount) { /* ... */ }
```

**Purpose**: Maintain aggregate boundary and consistency

**Related**: 
- [Wallet Aggregate](./aggregate.md) - Aggregate boundary definition

---

## Operational Business Rules

### BR-W-013: Reserve-Release Compensation
**Rule**: Reserved funds must be released back to available balance on service failure.

**Formal Definition**: On failure, execute `ReleaseReserve(amount)` to compensate `Reserve(amount)`

**Purpose**: Prevent fund lockup on failed operations

**Enforcement**: Application-level compensation logic in use cases

**Implementation Pattern**:
```csharp
// In application layer or use case
try
{
    wallet.Reserve(amount);
    // ... perform service operation
    wallet.ConfirmDeduction(amount);
}
catch (Exception)
{
    wallet.ReleaseReserve(amount); // Compensation
    throw;
}
```

**Domain Events**:
- `Reserve` publishes `BalanceChangedDomainEvent` with `BalanceOperation.Reserve`
- `ReleaseReserve` publishes `BalanceChangedDomainEvent` with `BalanceOperation.ReleaseReserve`

**Related**: 
- [Application use cases](../../application/use-cases/) - Compensation patterns

---

### BR-W-014: Soft Delete Retention
**Rule**: Deleted wallets must be retained for 7 years for audit purposes.

**Formal Definition**: `DeletedAt` timestamp set, data retained for 7 years

**Purpose**: Financial compliance and audit trail

**Enforcement**:
- `SoftDelete()` method sets `DeletedAt = DateTime.UtcNow`
- Wallet marked as deleted via `IsDeleted` computed property
- Data retention managed by database policies (not domain logic)

**Implementation**:
```csharp
public void SoftDelete()
{
    DeletedAt = DateTime.UtcNow;
    AddLocalEvent(new WalletDeletedDomainEvent(Id, UserId));
}
```

**Computed Property**:
```csharp
public bool IsDeleted => DeletedAt.HasValue;
```

**Filtering in Repository**:
```csharp
// Active wallets filter
.Where(x => x.UserId == userId && !x.IsDeleted)
```

**Related**: 
- [Wallet Lifecycle](./lifecycle.md) - Soft delete process
- [Wallet Domain Events](./domain-events.md) - WalletDeletedDomainEvent

---

## Business Rule Enforcement Architecture

### Rule Pattern

All business rules follow the domain rule pattern:

```csharp
public abstract class BusinessRule
{
    public abstract string Message { get; }
    public abstract bool IsBroken();
}

// Example rule
public class SufficientBalanceRule : BusinessRule
{
    private readonly decimal _availableBalance;
    private readonly decimal _amount;

    public SufficientBalanceRule(decimal availableBalance, decimal amount)
    {
        _availableBalance = availableBalance;
        _amount = amount;
    }

    public override string Message => "Insufficient balance available";
    
    public override bool IsBroken() => _availableBalance < _amount;
}
```

### Rule Checking

Rules are checked in aggregate methods:

```csharp
public void Reserve(decimal amount)
{
    CheckRule(new SufficientBalanceRule(AvailableBalance, amount));
    CheckRule(new WalletNotDeletedRule(this));
    // ... perform operation if all rules pass
}
```

### Rule Catalog

**Balance Rules**:
- `SufficientBalanceRule` - Validates sufficient available balance
- `SufficientReservedBalanceRule` - Validates sufficient reserved balance
- `CreditAmountMustBePositiveRule` - Validates credit amount is positive
- `BalanceMustNotBeNegativeRule` - Validates balance non-negative

**Asset Rules**:
- `AssetAlreadyExistsRule` - Validates asset type uniqueness
- `AssetBalanceRule` - Validates sufficient asset balance
- `AssetReservedBalanceRule` - Validates sufficient asset reserved balance

**Wallet Rules**:
- `WalletNotDeletedRule` - Validates wallet is not deleted

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition and operations
- [Wallet Model](./model.md) - Entity and Value Object definitions
- [Wallet Lifecycle](./lifecycle.md) - State transitions
- [Wallet Domain Events](./domain-events.md) - Events published on operations
- [Wallet Repositories](./repositories.md) - Database persistence rules