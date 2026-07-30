# Wallet Model

This document describes the Entities and Value Objects that comprise the Wallet Aggregate from a **domain perspective**. For technical implementation details including database mapping and audit fields, see [Technical Implementation](#technical-implementation) below.

## Domain Model

### Wallet (Aggregate Root)

**Purpose**: User wallet with balance tracking

**Base Class**: `BillingAggregateRoot` (extends `TMTFullAuditedAggregateRoot<long>`)

**Domain Properties**:

```csharp
public class Wallet : BillingAggregateRoot
{
    // Core domain properties
    public string UserId { get; set; }
    public decimal AvailableBalance { get; private set; }
    public decimal ReservedBalance { get; private set; }
    public string Currency { get; private set; }
    public List<WalletAsset> Assets { get; private set; }
    
    // Domain computed property
    public decimal TotalBalance => AvailableBalance + ReservedBalance;
}
```

**Domain Property Descriptions**:
- `UserId` - **string** - Unique user identifier (business key, UNIQUE)
- `AvailableBalance` - **decimal** - Available funds for consumption (private setter, domain-protected)
- `ReservedBalance` - **decimal** - Funds locked for pending transactions (private setter, domain-protected)
- `Currency` - **string** - Currency type (default: "VND", private setter, domain-protected)
- `Assets` - **List<WalletAsset>** - Collection of wallet assets (domain-managed)
- `TotalBalance` - **decimal** - Computed property (available + reserved, read-only)

**Key Operations**:

#### CreateNew(string userId, string currency = "VND")
- **Purpose**: Static factory method to create new wallet
- **Returns**: New `Wallet` instance
- **Publishes**: `WalletCreatedDomainEvent`

#### Reserve(decimal amount)
- **Purpose**: Reserve funds for pending transaction
- **Rules**: 
  - Must have sufficient available balance
  - Wallet must not be deleted
- **Publishes**: `BalanceChangedDomainEvent` with `BalanceOperation.Reserve`

#### ConfirmDeduction(decimal amount)
- **Purpose**: Confirm deduction after service completion
- **Rules**: 
  - Must have sufficient reserved balance
  - Wallet must not be deleted
- **Publishes**: `BalanceChangedDomainEvent` with `BalanceOperation.ConfirmDeduction`

#### ReleaseReserve(decimal amount)
- **Purpose**: Release reserved funds on failure
- **Rules**: 
  - Must have sufficient reserved balance
  - Wallet must not be deleted
- **Publishes**: `BalanceChangedDomainEvent` with `BalanceOperation.ReleaseReserve`

#### Credit(decimal amount)
- **Purpose**: Add funds to available balance
- **Rules**: 
  - Amount must be positive
  - Wallet must not be deleted
- **Publishes**: `BalanceChangedDomainEvent` with `BalanceOperation.Credit`

#### AddAsset(AssetType assetType, decimal initialBalance, DateTime? expiresAt)
- **Purpose**: Add new asset type to wallet
- **Rules**: Asset type must not already exist
- **Action**: Creates `WalletAsset` and optionally credits initial balance

#### SoftDelete()
- **Purpose**: Soft delete wallet
- **Action**: Sets inherited audit fields `IsDeleted = true` and `DeletionTime = DateTime.UtcNow`
- **Publishes**: `WalletDeletedDomainEvent`

#### HasSufficientBalance(decimal amount)
- **Purpose**: Check if wallet has sufficient balance
- **Returns**: `true` if not deleted and `AvailableBalance >= amount`

**Business Rules Enforced**:
- Available balance cannot be negative: `AvailableBalance >= 0`
- Reserved balance cannot be negative: `ReservedBalance >= 0`
- Total balance: `TotalBalance = AvailableBalance + ReservedBalance`
- Cannot operate on deleted wallet

---

### WalletAsset

**Purpose**: Multi-asset support within a wallet

**Base Class**: `Entity<Guid>` from ABP Framework

**Properties**:

```csharp
public class WalletAsset : Entity<Guid>
{
    public Guid WalletId { get; private set; }
    public AssetType AssetType { get; private set; }
    public decimal Balance { get; private set; }
    public decimal ReservedBalance { get; private set; }
    public DateTime? ExpiresAt { get; private set; }
    
    // Computed property
    public bool IsExpired => ExpiresAt.HasValue && DateTime.UtcNow > ExpiresAt.Value;
}
```

**Property Descriptions**:
- `Id` (inherited) - `Guid` - Unique asset identifier (PRIMARY KEY)
- `WalletId` - `Guid` - Foreign key to Wallet
- `AssetType` - `AssetType` - Asset classification (Value Object)
- `Balance` - `decimal` - Asset-specific available balance (private setter)
- `ReservedBalance` - `decimal` - Asset-specific reserved balance (private setter)
- `ExpiresAt` - `DateTime?` - Expiration date (null if never expires)
- `IsExpired` - `bool` - Computed expiration status

**Key Operations** (Internal - used by Wallet aggregate):

#### Reserve(decimal amount)
- **Purpose**: Reserve asset funds
- **Rules**: Must have sufficient balance
- **Action**: Decreases balance, increases reserved balance

#### ConfirmDeduction(decimal amount)
- **Purpose**: Confirm deduction from reserved funds
- **Rules**: Must have sufficient reserved balance
- **Action**: Decreases reserved balance

#### ReleaseReserve(decimal amount)
- **Purpose**: Release reserved funds back to available
- **Rules**: Must have sufficient reserved balance
- **Action**: Decreases reserved balance, increases balance

#### Credit(decimal amount)
- **Purpose**: Add funds to asset balance
- **Rules**: Amount must be positive
- **Action**: Increases balance

#### IsValidForExpirationCheck()
- **Purpose**: Check if asset should be evaluated for expiration
- **Returns**: `true` if asset type can expire and is not yet expired

**Constraints**:
- One asset per type per wallet: Enforced by database unique constraint
- Balance cannot be negative: Enforced by `AssetBalanceRule`
- Reserved balance cannot be negative: Enforced by `AssetReservedBalanceRule`

**Lifecycle**:
- Created when wallet receives specific asset type via `AddAsset()`
- Expires if `ExpiresAt` is set and current time > `ExpiresAt`
- Soft deleted when parent wallet is deleted

---

## Value Objects

### AssetType

**Purpose**: Asset classification Value Object

**Base Class**: `ValueObject` from domain building blocks

**Properties**:

```csharp
public class AssetType : ValueObject
{
    public string Code { get; }
    public string Name { get; }
    public bool CanExpire { get; }
}
```

**Property Descriptions**:
- `Code` - `string` - Unique asset type code
- `Name` - `string` - Human-readable asset type name
- `CanExpire` - `bool` - Whether this asset type supports expiration

**Static Factory Methods**:
- `WiCredit()` - Standard WION credits (never expires)
- `Promotion()` - Promotional credits (can expire)
- `Gift()` - Gift credits from campaigns (can expire)
- `Trial()` - Trial credits for new users (can expire)
- `AiToken()` - AI service tokens (never expires)

**Factory Method**:
- `Of(string code)` - Create `AssetType` from code string

**Asset Type Catalog**:

| Code | Name | Can Expire | Description |
|-----|-----|------------|-------------|
| WI_CREDIT | Wi Credit | No | Standard WION internal payment unit (1 VNĐ = 1 Wi Credit) |
| PROMOTION | Promotion | Yes | Promotional credits with expiration |
| GIFT | Gift | Yes | Gift credits from campaigns |
| TRIAL | Trial | Yes | Trial credits for new users |
| AI_TOKEN | AI Token | No | AI service tokens (future) |

**Operations**:
- Value object equality based on `Code`
- Immutable (all properties are private set)
- Thread-safe for read operations

**Asset Type Rules**:
- `WI_CREDIT` - Never expires, base currency unit
- `PROMOTION` - Can expire, typically for time-limited offers
- `GIFT` - Can expire, campaign-based credits
- `TRIAL` - Can expire, new user onboarding credits
- `AI_TOKEN` - Never expires, specific service tokens

---

## Relationships

### Entity Relationship Diagram

```
Wallet (Aggregate Root)
├── Id: Guid (PK)
├── UserId: string (Unique)
├── AvailableBalance: decimal
├── ReservedBalance: decimal
├── Currency: string
├── DeletedAt: DateTime?
└── Assets: List<WalletAsset> (Owned Collection)
    ├── Id: Guid (PK)
    ├── WalletId: Guid (FK → Wallet.Id)
    ├── AssetType: AssetType (Value Object)
    ├── Balance: decimal
    ├── ReservedBalance: decimal
    └── ExpiresAt: DateTime?
```

### Relationship Rules

**Wallet → WalletAsset** (One-to-Many):
- One wallet can have multiple assets
- Assets are owned entities (share same table or separate table with FK)
- Cascade delete: When wallet is deleted, assets are automatically deleted
- Asset operations are internal to the Wallet aggregate

**Wallet → Tenant** (Many-to-One):
- Each wallet belongs to exactly one tenant
- `UserId` is unique (one wallet per tenant)
- Enforced by database unique constraint

### Aggregate Boundary

**Wallet is the Aggregate Root**:
- All operations go through Wallet entity
- WalletAsset cannot be accessed directly outside the aggregate
- Wallet maintains consistency of its assets
- External repositories only work with Wallet aggregate

---

## Database Mapping

### Entity Framework Core Configuration

**DbContext**: `BillingDbContext`

**DbSet**: `DbSet<Wallet> Wallets`

**Table Configuration**:

```csharp
builder.Entity<Wallet>(b =>
{
    // Table mapping
    b.ToTable(BillingConsts.DbTablePrefix + "Wallets", BillingConsts.DbSchema);
    
    // Primary key
    b.HasKey(x => x.Id);
    
    // Domain property configurations
    b.Property(x => x.Id).ValueGeneratedOnAdd();
    b.Property(x => x.UserId).IsRequired().HasMaxLength(BillingConsts.UserIdMaxLength);
    b.Property(x => x.Currency).IsRequired().HasMaxLength(BillingConsts.CurrencyMaxLength);
    b.Property(x => x.AvailableBalance).HasColumnType("decimal(18,2)");
    b.Property(x => x.ReservedBalance).HasColumnType("decimal(18,2)");
    
    // Indexes for domain properties
    b.HasIndex(x => x.UserId).IsUnique();
    b.HasIndex(x => x.IsDeleted);
    
    // Note: Audit fields (CreationTime, CreatorId, IsDeleted, DeletionTime, etc.) 
    // are automatically configured by the TMT framework base class
    
    // Owned collection for assets
    b.OwnsMany(x => x.Assets, asset =>
    {
        asset.WithOwner().HasForeignKey("WalletId");
        asset.Property("AssetTypeCode")
            .IsRequired()
            .HasMaxLength(50);
        asset.Property("Balance")
            .HasColumnType("decimal(18,2)");
        asset.Property("ReservedBalance")
            .HasColumnType("decimal(18,2)");
        asset.Property("ExpiresAt")
            .IsRequired(false);
        
        asset.HasIndex("AssetTypeCode", "WalletId").IsUnique();
    });
});
```

### Database Schema

**Wallets Table**:

```sql
CREATE TABLE Billing.Wallets (
    -- Primary key
    Id BIGINT NOT NULL PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    
    -- Domain properties
    UserId NVARCHAR(256) NOT NULL UNIQUE,
    AvailableBalance DECIMAL(18,2) NOT NULL DEFAULT 0,
    ReservedBalance DECIMAL(18,2) NOT NULL DEFAULT 0,
    Currency NVARCHAR(10) NOT NULL,
    
    -- Audit fields (inherited from TMTFullAuditedAggregateRoot)
    CreationTime TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CreatorId TEXT NULL,
    CreateTransactionId TEXT NULL,
    LastModificationTime TIMESTAMP NULL,
    LastModifierId TEXT NULL,
    LastModifierTransactionId TEXT NULL,
    IsDeleted BIT NOT NULL DEFAULT 0,
    DeleterId TEXT NULL,
    DeletionTime TIMESTAMP NULL,
    DeleteTransactionId TEXT NULL,
    
    -- Additional framework fields
    ExtraProperties TEXT NULL,
    ConcurrencyStamp NVARCHAR(40) NULL,
    
    -- Constraints
    CONSTRAINT UQ_Wallets_UserId UNIQUE (UserId),
    CONSTRAINT CK_Wallets_AvailableBalance CHECK (AvailableBalance >= 0),
    CONSTRAINT CK_Wallets_ReservedBalance CHECK (ReservedBalance >= 0)
);

-- Indexes
CREATE INDEX IX_Wallets_UserId ON Billing.Wallets(UserId);
CREATE INDEX IX_Wallets_IsDeleted ON Billing.Wallets(IsDeleted);
CREATE INDEX IX_Wallets_CreationTime ON Billing.Wallets(CreationTime);
```
    -- Constraints
    CONSTRAINT UQ_Wallets_UserId UNIQUE (UserId),
    CONSTRAINT CK_Wallets_AvailableBalance CHECK (AvailableBalance >= 0),
    CONSTRAINT CK_Wallets_ReservedBalance CHECK (ReservedBalance >= 0)
);

-- Indexes
CREATE INDEX IX_Wallets_UserId ON Billing.Wallets(UserId);
CREATE INDEX IX_Wallets_IsDeleted ON Billing.Wallets(IsDeleted);
```

**WalletAssets Table** (Owned Entity):

```sql
CREATE TABLE Billing.WalletAssets (
    -- Primary key
    Id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    
    -- Foreign key
    WalletId UNIQUEIDENTIFIER NOT NULL,
    
    -- Asset properties
    AssetTypeCode NVARCHAR(50) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    ReservedBalance DECIMAL(18,2) NOT NULL DEFAULT 0,
    ExpiresAt DATETIME2 NULL,
    
    -- Constraints
    CONSTRAINT FK_WalletAssets_Wallets FOREIGN KEY (WalletId) 
        REFERENCES Billing.Wallets(Id) ON DELETE CASCADE,
    CONSTRAINT UQ_WalletAssets_Type UNIQUE (AssetTypeCode, WalletId),
    CONSTRAINT CK_WalletAssets_Balance CHECK (Balance >= 0),
    CONSTRAINT CK_WalletAssets_ReservedBalance CHECK (ReservedBalance >= 0)
);

-- Indexes
CREATE INDEX IX_WalletAssets_WalletId ON Billing.WalletAssets(WalletId);
CREATE INDEX IX_WalletAssets_AssetTypeCode ON Billing.WalletAssets(AssetTypeCode);
```

---

## Invariants and Business Rules

### Wallet Invariants

1. **Balance Non-Negative**: `AvailableBalance >= 0` and `ReservedBalance >= 0`
2. **Total Balance**: `TotalBalance = AvailableBalance + ReservedBalance`
3. **Deleted Wallet Operations**: Cannot modify deleted wallet balances
4. **Sufficient Balance**: Cannot reserve more than available balance
5. **Sufficient Reserve**: Cannot confirm deduction more than reserved balance

### WalletAsset Invariants

1. **Balance Non-Negative**: `Balance >= 0` and `ReservedBalance >= 0`
2. **Unique Asset Type**: One asset per type per wallet
3. **Expiration Check**: Assets expire when `ExpiresAt < DateTime.UtcNow`
4. **Asset Type Validation**: Must be valid predefined asset type

### Domain Events

All balance operations publish domain events:
- `WalletCreatedDomainEvent` - New wallet created
- `WalletDeletedDomainEvent` - Wallet soft deleted
- `BalanceChangedDomainEvent` - Balance modified (with operation type)

---

## Technical Implementation

This section documents the **technical infrastructure** provided by the TMT Framework that supports the Wallet domain model.

### Framework Base Class

**Base Class**: `TMTFullAuditedAggregateRoot<long>`

**Purpose**: Provides automatic audit trail, concurrency control, and soft-delete functionality

### Technical Properties (Framework-Provided)

The `Wallet` entity inherits these **technical infrastructure properties** from `TMTFullAuditedAggregateRoot<long>`:

#### Audit Fields
```csharp
// Inherited from TMTFullAuditedAggregateRoot<long>
public bool IsDeleted { get; set; }                    // Soft delete flag
public string DeleterId { get; set; }                   // Who deleted (technical)
public DateTime? DeletionTime { get; set; }            // When deleted (technical)
public string DeleteTransactionId { get; set; }         // Which transaction (technical)

public DateTime? LastModificationTime { get; set; }      // Last modified (technical)
public string LastModifierId { get; set; }              // Who modified (technical)
public string LastModifierTransactionId { get; set; }  // Which transaction (technical)

public DateTime CreationTime { get; set; }               // When created (technical)
public string CreatorId { get; set; }                    // Who created (technical)
public string CreateTransactionId { get; set; }          // Which transaction (technical)
```

#### Concurrency Control
```csharp
public string ConcurrencyStamp { get; set; }             // Optimistic concurrency
```

#### Extension Properties
```csharp
public string ExtraProperties { get; set; }             // Flexible JSON storage
```

### Technical Behavior

#### Automatic Data Filtering (ABP Framework)
The ABP Framework provides **automatic data filtering** through `IDataFilter`:

**Soft Delete Filtering** (Automatic):
- `IDataFilter<ISoftDelete>` automatically filters queries where `IsDeleted = false`
- **No manual filtering needed** in repository queries
- Example:
  ```csharp
  // Framework automatically adds: WHERE IsDeleted = false
  var wallet = await _walletRepository.FindByUserIdAsync("user-123");
  
  // To include deleted items, explicitly bypass:
  var allWallets = await _repository
      .WithDetailsAsync(x => x.Assets)
      .IgnoreQueryFilters(); // Bypasses IDataFilter
  ```

**Multi-Tenancy Filtering** (Not Used):
- Wallet does **NOT** implement `IMultiTenant` 
- No automatic tenant filtering applied
- Uses `UserId` as the business key instead

#### Automatic Population
The ABP Framework automatically populates audit fields:
- `CreationTime` = `DateTime.UtcNow` on insert
- `CreatorId` = current user on insert  
- `LastModificationTime` = `DateTime.UtcNow` on update
- `LastModifierId` = current user on update
- Framework `ICurrentUser` and `ICurrentTransaction` provide context

#### Soft Delete Implementation
```csharp
// Domain operation uses technical infrastructure
public void SoftDelete()
{
    // Domain logic: mark as deleted
    // Framework's IDataFilter will automatically exclude this from future queries
    IsDeleted = true;              
    DeletionTime = DateTime.UtcNow; 
    
    // Domain event (domain-relevant data)
    AddLocalEvent(new WalletDeletedDomainEvent(Id, UserId));
}
```

#### Query Behavior Examples
```csharp
// ABP Framework automatically adds: WHERE IsDeleted = false
public async Task<Wallet> FindByUserIdAsync(string userId)
{
    // Framework query: SELECT * FROM Wallets WHERE UserId = @userId AND IsDeleted = false
    return await (await GetQueryableAsync())
        .Include(x => x.Assets)
        .Where(x => x.UserId == userId)  // Only need business condition!
        .FirstOrDefaultAsync();
}

// To get deleted wallets, explicitly bypass framework filter
public async Task<List<Wallet>> GetDeletedWalletsAsync()
{
    return await (await GetQueryableAsync())
        .Include(x => x.Assets)
        .IgnoreQueryFilters()  // Bypasses IDataFilter soft delete filter
        .Where(x => x.IsDeleted == true) // Manual filtering when bypassing
        .ToListAsync();
}

### Database Mapping

The TMT Framework automatically maps these technical fields to the database schema. See [Database Schema](#database-schema) for complete table structure.

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition and operations
- [Wallet Business Rules](./business-rules.md) - Business invariants
- [Wallet Domain Events](./domain-events.md) - Events published by Wallet
- [Wallet Repositories](./repositories.md) - Repository interfaces