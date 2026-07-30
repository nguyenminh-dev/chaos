# Wallet Repositories

This document defines the repository interfaces for the Wallet Aggregate. Repositories mediate between the Domain and Infrastructure layers.

## Repository Interfaces

### IWalletRepository

**Purpose**: Manage Wallet Aggregate persistence

**Interface Definition**:
```csharp
public interface IWalletRepository : IRepository<Wallet>
{
    Task<Wallet> FindByUserIdAsync(string userId);
    Task<Wallet> FindByIdAsync(Guid walletId);
    Task<bool> ExistsByUserIdAsync(string userId);
    Task<List<Wallet>> GetActiveWalletsAsync();
    Task<List<Wallet>> GetDeletedWalletsAsync();
}
```

**Base Interface**: Extends `IRepository<Wallet>` from ABP Framework, providing standard CRUD operations.

**Methods**:

#### FindByUserIdAsync(string userId)
- **Purpose**: Load wallet by user ID
- **Returns**: `Task<Wallet>` - Wallet or null if not found
- **Implementation**: 
  - Queries database with `Include(x => x.Assets)` to load wallet assets
  - Filters by `userId` only (ABP Framework `IDataFilter` automatically adds `IsDeleted = false`)
  - Uses Entity Framework Core async operations
- **Throws**: Database exception on query failure

#### FindByIdAsync(Guid walletId)
- **Purpose**: Load wallet by wallet ID
- **Returns**: `Task<Wallet>` - Wallet or null if not found
- **Implementation**:
  - Queries database with `Include(x => x.Assets)` to load wallet assets
  - Filters by `walletId` only (ABP Framework `IDataFilter` automatically adds `IsDeleted = false`)
  - Uses Entity Framework Core async operations
- **Throws**: Database exception on query failure

#### ExistsByUserIdAsync(string userId)
- **Purpose**: Check if wallet exists for user
- **Returns**: `Task<bool>` - True if wallet exists and is not deleted
- **Implementation**: Uses `AnyAsync()` with filter on `userId` only (ABP Framework `IDataFilter` automatically handles soft delete)
- **Performance**: Optimized query that doesn't load entity data

#### GetActiveWalletsAsync()
- **Purpose**: Get all non-deleted wallets
- **Returns**: `Task<List<Wallet>>` - List of active wallets with assets
- **Implementation**: 
  - Includes wallet assets via `Include(x => x.Assets)`
  - **ABP Framework `IDataFilter` automatically filters `IsDeleted = false`**
  - Uses index on `IsDeleted` column for performance
- **Performance**: Suitable for admin/monitoring operations

#### GetDeletedWalletsAsync()
- **Purpose**: Get all soft-deleted wallets
- **Returns**: `Task<List<Wallet>>` - List of deleted wallets
- **Implementation**: 
  - Must use `IgnoreQueryFilters()` to bypass ABP Framework's `IDataFilter`
  - Then manually filters by `IsDeleted == true`
- **Use Case**: Audit trails, recovery operations

---

## Implementation Notes

### Infrastructure Layer

**Implementation**: `EfCoreWalletRepository`

**Location**: `Wion.Billing.EntityFrameworkCore/Wallets/EfCoreWalletRepository.cs`

**Base Class**: Extends `EfCoreRepository<BillingDbContext, Wallet>`

**Key Features**:
- Uses ABP Framework's Entity Framework Core repository pattern
- Implements async/await for all database operations
- Follows C# naming conventions (`Async` suffix)
- Leverages EF Core's `Include()` for eager loading

### Database Context

**DbContext**: `BillingDbContext`

**DbSet**: `DbSet<Wallet> Wallets`

**Table Configuration**:
```csharp
builder.Entity<Wallet>(b =>
{
    b.ToTable(BillingConsts.DbTablePrefix + "Wallets", BillingConsts.DbSchema);
    
    // Configure primary key
    b.HasKey(x => x.Id);
    
    // Configure properties
    b.Property(x => x.UserId).IsRequired().HasMaxLength(BillingConsts.UserIdMaxLength);
    b.Property(x => x.Currency).IsRequired().HasMaxLength(BillingConsts.CurrencyMaxLength);
    b.Property(x => x.AvailableBalance).HasColumnType("decimal(18,2)");
    b.Property(x => x.ReservedBalance).HasColumnType("decimal(18,2)");
    b.Property(x => x.DeletedAt).IsRequired(false);
    
    // Indexes
    b.HasIndex(x => x.UserId).IsUnique();
    b.HasIndex(x => x.IsDeleted);
    
    // Owned collection for assets
    b.OwnsMany(x => x.Assets, asset => { /* ... */ });
});
```

### Database Schema

**Wallet Table**:
```sql
CREATE TABLE Billing.Wallets (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    UserId NVARCHAR(255) NOT NULL UNIQUE,
    AvailableBalance DECIMAL(18,2) NOT NULL DEFAULT 0,
    ReservedBalance DECIMAL(18,2) NOT NULL DEFAULT 0,
    Currency NVARCHAR(50) NOT NULL,
    DeletedAt DATETIME2 NULL,
    -- ABP Framework standard fields
    CreationTime DATETIME2 NOT NULL,
    CreatorId UNIQUEIDENTIFIER NULL,
    LastModificationTime DATETIME2 NULL,
    LastModifierId UNIQUEIDENTIFIER NULL,
    IsDeleted BIT NOT NULL DEFAULT 0,
    
    CONSTRAINT UQ_UserId UNIQUE (UserId)
);

-- Indexes
CREATE INDEX IX_Wallets_UserId ON Billing.Wallets(UserId);
CREATE INDEX IX_Wallets_IsDeleted ON Billing.Wallets(IsDeleted);
```

**WalletAssets Table** (Owned Entity):
```sql
CREATE TABLE Billing.WalletAssets (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    WalletId UNIQUEIDENTIFIER NOT NULL,
    AssetTypeCode NVARCHAR(50) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    ReservedBalance DECIMAL(18,2) NOT NULL DEFAULT 0,
    ExpiresAt DATETIME2 NULL,
    
    CONSTRAINT FK_WalletAssets_Wallets FOREIGN KEY (WalletId) 
        REFERENCES Billing.Wallets(Id) ON DELETE CASCADE,
    CONSTRAINT UQ_WalletAssets_Type UNIQUE (AssetTypeCode, WalletId)
);
```

### Database Queries

**ABP Framework Automatic Filtering**:
The ABP Framework's `IDataFilter<ISoftDelete>` automatically adds `WHERE IsDeleted = 0` to all queries.

**Standard Queries** (with automatic filtering):

```sql
-- Find wallet by user ID (with assets)
-- ABP Framework automatically adds: AND w.IsDeleted = 0
SELECT w.*, a.*
FROM Billing.Wallets w
LEFT JOIN Billing.WalletAssets a ON w.Id = a.WalletId
WHERE w.UserId = @userId;

-- Find wallet by ID (with assets)
-- ABP Framework automatically adds: AND w.IsDeleted = 0
SELECT w.*, a.*
FROM Billing.Wallets w
LEFT JOIN Billing.WalletAssets a ON w.Id = a.WalletId
WHERE w.Id = @walletId;

-- Check existence (active wallets only)
-- ABP Framework automatically adds: AND IsDeleted = 0
SELECT CASE WHEN EXISTS (
    SELECT 1 FROM Billing.Wallets 
    WHERE UserId = @userId
) THEN 1 ELSE 0 END;
```

**Manual Queries** (when bypassing framework filter):

```sql
-- Find deleted wallets (requires manual filtering)
-- Must use IgnoreQueryFilters() then manually add condition
SELECT w.*, a.*
FROM Billing.Wallets w
LEFT JOIN Billing.WalletAssets a ON w.Id = a.WalletId
WHERE w.IsDeleted = 1;

-- All wallets (including deleted)
-- Must use IgnoreQueryFilters() to bypass automatic filter
SELECT w.*, a.*
FROM Billing.Wallets w
LEFT JOIN Billing.WalletAssets a ON w.Id = a.WalletId;
```

### Performance Considerations

**Indexes**:
- Unique index on `UserId` for fast tenant-based lookups
- Index on `IsDeleted` for filtering active/deleted wallets
- Composite unique index on `(AssetTypeCode, WalletId)` for asset lookups

**Query Optimization**:
- Use `Include()` for eager loading related assets
- Filter at database level with `WHERE` clauses
- Use async operations to avoid thread blocking
- Leverage existing indexes for filtering operations

---

## Transaction Management

### Unit of Work Pattern

ABP Framework provides automatic Unit of Work management:

```csharp
public class WalletApplicationService : ApplicationService
{
    private readonly IWalletRepository _walletRepository;
    
    [UnitOfWork] // Automatic transaction management
    public async Task CreditBalanceAsync(string userId, decimal amount)
    {
        var wallet = await _walletRepository.FindByUserIdAsync(userId);
        wallet.Credit(amount);
        
        // Automatically commits transaction on method completion
        await _walletRepository.UpdateAsync(wallet);
    }
}
```

### Transaction Boundary

**Wallet Aggregate is the transaction boundary**:
- All operations on a single wallet in one transaction
- Never modify multiple wallets in one transaction
- Cross-wallet operations via Domain Events with eventual consistency
- ABP Framework automatically manages transactions around application service methods

### Database Connection

**Connection String**: `Default` connection from appsettings.json

**Transaction Isolation**: Read Committed (ABP Framework default)

**Retry Policy**: Configured in ABP Framework options for transient failures

---

## Error Handling

### Repository Exceptions

ABP Framework provides built-in exception handling:

| Error Type | When Thrown | HTTP Code |
|------------|-------------|-----------|
| EntityNotFoundException | Entity not found | 404 |
| AbpDbConcurrencyException | Optimistic lock failure | 409 |
| DbUpdateException | Database constraint violation | 400/409 |
| AbpException | General ABP exceptions | 500 |

### Business Rule Exceptions

Domain rules throw `BusinessRuleException`:

```csharp
// Example: Insufficient balance
throw new BusinessRuleException("Insufficient balance available");

// Example: Wallet is deleted
throw new BusinessRuleException("Cannot operate on deleted wallet");
```

### Error Recovery

**Automatic Retry**: ABP Framework handles transient database errors

**Validation**: Business rules validate before database operations

**Transaction Rollback**: Automatic on unhandled exceptions

---

## ABP Framework Integration

### Repository Pattern

**Base Repository**: `IRepository<TEntity, TKey>` from ABP Framework

**Available Methods** (inherited from base repository):
- `GetAsync(TKey id)` - Get entity by ID
- `InsertAsync(TEntity entity)` - Insert new entity
- `UpdateAsync(TEntity entity)` - Update existing entity
- `DeleteAsync(TEntity entity)` - Delete entity
- `ListAsync()` - Get all entities

**Custom Methods**: Added domain-specific queries in `IWalletRepository`

### Dependency Injection

**Registration**: Automatic in ABP Framework

**Usage**:
```csharp
public class WalletApplicationService : ApplicationService
{
    private readonly IWalletRepository _walletRepository;
    
    public WalletApplicationService(IWalletRepository walletRepository)
    {
        _walletRepository = walletRepository;
    }
}
```

### Caching

**ABP Caching**: Can be enabled via ABP Framework caching attributes

**Example** (optional):
```csharp
[Cache(AbsoluteExpirationRelativeToNow = 60)] // 60 seconds
public async Task<Wallet> GetByUserIdAsync(string userId)
{
    return await _walletRepository.FindByUserIdAsync(userId);
}
```

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition
- [Wallet Model](./model.md) - Entity definitions
- [Wallet Business Rules](./business-rules.md) - Business invariants enforced by repositories
- [ABP Framework Repository Documentation](https://docs.abp.io/en/abp/latest/Repositories)