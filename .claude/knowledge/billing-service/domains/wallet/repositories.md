# Wallet Repositories

This document defines the repository interfaces for the Wallet Aggregate. Repositories mediate between the Domain and Infrastructure layers.

## Repository Interfaces

### IWalletRepository

**Purpose**: Manage Wallet Aggregate persistence

**Interface Definition**:
```typescript
interface IWalletRepository {
  // Load operations
  findById(tenantId: string): Promise<Wallet | null>;
  findByTenantId(tenantId: string): Promise<Wallet | null>;
  
  // Query operations
  findAll(spec: Specification): Promise<Wallet[]>;
  findActiveWallets(): Promise<Wallet[]>;
  findDeletedWallets(): Promise<Wallet[]>;
  
  // Persistence operations
  save(wallet: Wallet): Promise<void>;
  update(wallet: Wallet): Promise<void>;
  delete(tenantId: string): Promise<void>;
  
  // Existence checks
  exists(tenantId: string): Promise<boolean>;
  existsActive(tenantId: string): Promise<boolean>;
}
```

**Methods**:

#### findById(tenantId: string)
- **Purpose**: Load wallet by tenant ID
- **Returns**: Wallet or null if not found
- **Throws**: RepositoryError on database failure
- **Cache**: Uses Redis cache with 60s TTL

#### findByTenantId(tenantId: string)
- **Purpose**: Alias for findById (semantic clarity)
- **Returns**: Wallet or null if not found

#### findAll(spec: Specification)
- **Purpose**: Query wallets with specification
- **Returns**: Array of Wallet matching specification
- **Example**: Find all wallets with balance > threshold

#### findActiveWallets()
- **Purpose**: Find all non-deleted wallets
- **Returns**: Array of active Wallet
- **Optimization**: Uses index on `is_deleted` column

#### save(wallet: Wallet)
- **Purpose**: Persist new wallet
- **Precondition**: Wallet must not already exist
- **Throws**: DuplicateKeyError if wallet exists
- **Side Effect**: Invalidates cache

#### update(wallet: Wallet)
- **Purpose**: Update existing wallet
- **Precondition**: Wallet must exist
- **Optimistic Locking**: Uses version field
- **Throws**: 
  - NotFoundError if wallet doesn't exist
  - ConcurrencyError if version mismatch
- **Side Effect**: Invalidates cache

#### delete(tenantId: string)
- **Purpose**: Soft delete wallet
- **Action**: Sets `isDeleted = true`
- **Throws**: NotFoundError if wallet doesn't exist
- **Side Effect**: Invalidates cache

---

### IWalletAssetRepository

**Purpose**: Manage WalletAsset persistence

**Interface Definition**:
```typescript
interface IWalletAssetRepository {
  // Load operations
  findById(id: string): Promise<WalletAsset | null>;
  findByTenantAndType(tenantId: string, assetType: AssetType): Promise<WalletAsset | null>;
  findByTenantId(tenantId: string): Promise<WalletAsset[]>;
  
  // Query operations
  findExpired(): Promise<WalletAsset[]>;
  findExpiringSoon(withinDays: number): Promise<WalletAsset[]>;
  
  // Persistence operations
  save(asset: WalletAsset): Promise<void>;
  update(asset: WalletAsset): Promise<void>;
  delete(id: string): Promise<void>;
  
  // Existence checks
  exists(tenantId: string, assetType: AssetType): Promise<boolean>;
}
```

**Methods**:

#### findByTenantAndType(tenantId: string, assetType: AssetType)
- **Purpose**: Load specific asset for tenant
- **Returns**: WalletAsset or null
- **Uniqueness**: Enforces one asset per type per tenant

#### findExpired()
- **Purpose**: Find all expired assets
- **Returns**: Array of expired WalletAsset
- **Criteria**: `expiresAt < NOW() AND isDeleted = false`

#### findExpiringSoon(withinDays: number)
- **Purpose**: Find assets expiring within N days
- **Returns**: Array of WalletAsset expiring soon
- **Criteria**: `expiresAt BETWEEN NOW() AND NOW() + N days`

#### save(asset: WalletAsset)
- **Purpose**: Persist new asset
- **Precondition**: Asset must not already exist for tenant+type
- **Throws**: DuplicateKeyError if asset exists

#### update(asset: WalletAsset)
- **Purpose**: Update existing asset
- **Optimistic Locking**: Uses version field
- **Throws**: ConcurrencyError on version mismatch

---

## Implementation Notes

### Infrastructure Layer

Repository interfaces are implemented in the Infrastructure layer:

```
Infrastructure/
└── Persistence/
    └── PostgreSQL/
        ├── WalletRepository.ts      # Implements IWalletRepository
        └── WalletAssetRepository.ts  # Implements IWalletAssetRepository
```

### Database Queries

**Key Queries**:

```sql
-- Find wallet by tenant ID (with cache)
SELECT * FROM wallet WHERE tenant_id = ? AND is_deleted = false;

-- Find active wallets
SELECT * FROM wallet WHERE is_deleted = false;

-- Find expired assets
SELECT * FROM wallet_asset 
WHERE expires_at < NOW() AND is_deleted = false;

-- Find asset by tenant and type
SELECT * FROM wallet_asset 
WHERE tenant_id = ? AND asset_type = ? AND is_deleted = false;
```

### Caching Strategy

**Cache Keys**:
- `wallet:{tenantId}` - Wallet data
- `wallet:asset:{tenantId}:{type}` - Asset data

**Cache Invalidation**:
- Delete on save/update/delete
- TTL: 60 seconds

**Cache-Aside Pattern**:
```typescript
async findById(tenantId: string): Promise<Wallet | null> {
  // 1. Check cache
  const cached = await cache.get(`wallet:${tenantId}`);
  if (cached) return JSON.parse(cached);
  
  // 2. Query database
  const wallet = await db.query('SELECT * FROM wallet WHERE tenant_id = ?', [tenantId]);
  
  // 3. Store in cache
  if (wallet) {
    await cache.set(`wallet:${tenantId}`, JSON.stringify(wallet), 'EX', 60);
  }
  
  return wallet;
}
```

---

## Transaction Management

### Unit of Work Pattern

Repositories participate in Unit of Work:

```typescript
async function consumeCredit(tenantId: string, amount: decimal) {
  await unitOfWork.begin();
  
  try {
    const wallet = await walletRepo.findById(tenantId);
    wallet.reserve(amount);
    
    await walletRepo.update(wallet);
    await ledgerRepo.createLedgerEntry(...);
    
    await unitOfWork.commit();
  } catch (error) {
    await unitOfWork.rollback();
    throw error;
  }
}
```

### Transaction Boundary

**Wallet Aggregate is the transaction boundary**:
- All operations on a single wallet in one transaction
- Never modify multiple wallets in one transaction
- Cross-wallet operations via Domain Events

---

## Error Handling

### Repository Exceptions

| Error Type | When Thrown | HTTP Code |
|------------|-------------|-----------|
| NotFoundError | Entity not found | 404 |
| DuplicateKeyError | Violates unique constraint | 409 |
| ConcurrencyError | Optimistic lock failure | 409 |
| RepositoryError | Database failure | 500 |

### Error Recovery

**Retry Strategy**:
- Transient errors: Retry 3 times with exponential backoff
- Non-transient errors: Fail immediately

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition
- [Wallet Model](./model.md) - Entity definitions
- [Wallet Business Rules](./business-rules.md) - Business invariants enforced by repositories
