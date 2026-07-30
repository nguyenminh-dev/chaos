# Wallet Management Use Case

**Purpose**: Create and manage tenant wallets with multi-asset support

**Implementation Status**: ✅ **COMPLETE**

---

## Business Goal

Enable automatic wallet creation for new tenants and provide wallet management operations for balance and asset management.

---

## Actor

**Primary Actors**:
- System (automatic wallet creation on tenant creation)
- Tenant Admin (manual wallet operations)

**Secondary Actors**:
- Billing Service (wallet operations)

---

## Trigger

**Automatic Creation**: TenantCreated event from Tenant Service

**Manual Operations**: Admin API requests for wallet operations

---

## Preconditions

**System State**:
- Valid tenant exists in system
- Tenant is active (not deleted)

**Wallet State** (for operations):
- Wallet exists for tenant
- Wallet is not deleted

---

## Main Flow

### Step 1: Automatic Wallet Creation

**Trigger**: Receive TenantCreated event from Tenant Service

**Action**: Create new wallet with default assets

**Input**:
```typescript
{
  tenantId: string,
  tenantName: string,
  currency: string // "VND"
}
```

**Processing**:
1. Validate tenant doesn't already have wallet
2. Create Wallet aggregate with id, tenantId, currency
3. Initialize with default Wi Credit asset (balance = 0)
4. Publish WalletCreated domain event
5. Persist wallet to database

**Output**:
```typescript
{
  walletId: long,
  tenantId: string,
  currency: string,
  availableBalance: 0,
  reservedBalance: 0,
  assets: [
    {
      assetType: "WI_CREDIT",
      balance: 0,
      reservedBalance: 0
    }
  ]
}
```

---

### Step 2: Asset Management

**Operation**: AddAsset(assetType, initialBalance, expiresAt)

**Input**:
```typescript
{
  walletId: long,
  assetType: AssetType, // "PROMOTION", "GIFT", "TRIAL", "AI_TOKEN"
  initialBalance: decimal,
  expiresAt: DateTime? (null for non-expiring assets)
}
```

**Processing**:
1. Load wallet by walletId
2. Validate asset type doesn't already exist
3. Create WalletAsset entity
4. If initialBalance > 0, credit wallet
5. Publish BalanceChanged domain event
6. Persist changes

**Output**: Updated wallet with new asset

---

### Step 3: Balance Operations

**Credit Operation**:
```typescript
Credit(walletId, amount)
```
- Validates amount > 0
- Updates availableBalance += amount
- Publishes BalanceChanged domain event

**Reserve Operation**:
```typescript
Reserve(walletId, amount)
```
- Validates sufficient available balance
- Updates availableBalance -= amount
- Updates reservedBalance += amount
- Publishes BalanceChanged domain event

**Confirm Deduction**:
```typescript
ConfirmDeduction(walletId, amount)
```
- Validates sufficient reserved balance
- Updates reservedBalance -= amount
- Publishes BalanceChanged domain event

**Release Reserve**:
```typescript
ReleaseReserve(walletId, amount)
```
- Validates sufficient reserved balance
- Updates reservedBalance -= amount
- Updates availableBalance += amount
- Publishes BalanceChanged domain event

---

## Alternative Flows

### Alternative 1: Wallet Already Exists
**Condition**: TenantCreated event received but wallet already exists

**Handling**:
- Log informational message
- Skip wallet creation
- Continue with existing wallet

**Reason**: Idempotency for tenant creation events

---

### Alternative 2: Asset Already Exists
**Condition**: AddAsset called with existing asset type

**Error**: AssetAlreadyExistsRule violation

**Response**:
```typescript
{
  success: false,
  error: {
    code: "ASSET_ALREADY_EXISTS",
    message: "Asset of type {assetType} already exists for wallet"
  }
}
```

---

### Alternative 3: Insufficient Balance
**Condition**: Reserve or ConfirmDeduction called with insufficient balance

**Error**: SufficientBalanceRule or SufficientReservedBalanceRule violation

**Response**:
```typescript
{
  success: false,
  error: {
    code: "INSUFFICIENT_BALANCE",
    message: "Insufficient balance for operation"
  }
}
```

---

## Error Flows

### Error 1: Invalid Credit Amount
**Condition**: Credit called with amount <= 0

**Error**: CreditAmountMustBePositiveRule violation

**Handling**:
- Return validation error
- No database operation
- No events published

**Response**:
```typescript
{
  success: false,
  error: {
    code: "VALIDATION_ERROR",
    message: "Credit amount must be positive"
  }
}
```

---

### Error 2: Wallet Not Found
**Condition**: Wallet operations requested for non-existent walletId

**Error**: EntityNotFoundException

**Handling**:
- Return 404 error
- No state changes
- No events published

**Response**:
```typescript
{
  success: false,
  error: {
    code: "WALLET_NOT_FOUND",
    message: "Wallet not found"
  }
}
```

---

### Error 3: Deleted Wallet Operation
**Condition**: Operation attempted on deleted wallet

**Error**: WalletNotDeletedRule violation

**Handling**:
- Return validation error
- No state changes
- No events published

**Response**:
```typescript
{
  success: false,
  error: {
    code: "WALLET_DELETED",
    message: "Cannot operate on deleted wallet"
  }
}
```

---

## Postconditions

**System State** (after successful wallet creation):
- New wallet exists for tenant
- Wallet has default Wi Credit asset
- WalletCreated event published

**System State** (after successful balance operation):
- Wallet balances updated
- BalanceChanged event published
- Database transaction committed

**Wallet State**:
- All business rules validated
- Domain invariants maintained
- Balance never negative (BR-W-002)
- Reserved balance never negative (BR-W-003)

**Side Effects**:
- Domain events published for downstream processing
- Database transaction committed
- Cache updated (if caching implemented)

---

## Aggregates Used

**Wallet Aggregate** (only aggregate involved):
- **Operations**: CreateNew, AddAsset, Credit, Reserve, ConfirmDeduction, ReleaseReserve
- **Business Rules Enforced**: All wallet business rules (BR-W-001 through BR-W-014)
- **Events Published**:
  - WalletCreated
  - BalanceChanged
  - WalletDeleted

**No other aggregates** - Use case operates within single aggregate boundary

---

## Dependencies

**Repository Dependencies**:
- `IWalletRepository` - Wallet persistence and retrieval

**External Service Dependencies**:
- `Tenant Service` - TenantCreated events (for automatic wallet creation)

**Domain Services**:
- None (all logic in Wallet aggregate)

---

## Performance Requirements

**Response Time**:
- Wallet creation: < 200ms (P95)
- Balance operations: < 100ms (P95)

**Throughput**:
- Support concurrent wallet operations per tenant
- Handle tenant creation bursts

**Resource Limits**:
- One wallet per tenant (BR-W-001)
- Maximum assets per wallet: 5 (WI_CREDIT, PROMOTION, GIFT, TRIAL, AI_TOKEN)

---

## Error Handling

**Validation Errors**:
- Invalid credit amount → 400 Bad Request
- Insufficient balance → 400 Bad Request
- Asset already exists → 409 Conflict

**Business Rule Violations**:
- Deleted wallet operation → 400 Bad Request
- Balance rule violation → 400 Bad Request

**System Errors**:
- Database connection error → 500 Internal Server Error
- Concurrent modification → 409 Conflict (retry)

---

## Security Considerations

**Authentication**:
- API Key authentication for service-to-service calls
- JWT authentication for admin operations

**Authorization**:
- System: Can create wallets automatically
- Tenant Admin: Can manage own tenant wallets
- Platform Admin: Can manage any wallet

**Tenant Isolation**:
- Multi-tenancy filter ensures data isolation
- Operations scoped to tenant context

**Audit Logging**:
- All wallet operations logged
- Balance changes tracked with audit trail

---

## Testing

**Unit Tests**:
1. **Should create wallet when**: New tenant created
2. **Should not duplicate wallet when**: TenantCreated event received twice
3. **Should add asset when**: Valid asset type provided
4. **Should reject asset when**: Asset type already exists
5. **Should credit balance when**: Valid amount provided
6. **Should reject credit when**: Amount <= 0
7. **Should reserve balance when**: Sufficient available balance
8. **Should reject reserve when**: Insufficient available balance

**Integration Tests**:
1. **Should create wallet end-to-end when**: TenantCreated event received
2. **Should complete balance operation when**: Valid request
3. **Should handle concurrent operations when**: Multiple operations on same wallet

**Domain Tests**:
- All business rules tested
- Domain events verified
- Aggregate boundaries maintained

---

## Implementation Details

**Application Service**: `WalletAppService`

**Key Methods**:
```csharp
public class WalletAppService : ApplicationService
{
    // Automatic wallet creation (event handler)
    public async Task HandleEventAsync(TenantCreatedEto eventData)
    {
        var wallet = Wallet.CreateNew(
            id: GuidGenerator.Create(),
            tenantId: eventData.TenantId,
            currency: "VND"
        );
        
        wallet.AddAsset(AssetType.WiCredit(), 0, null);
        
        await _walletRepository.InsertAsync(wallet);
    }
    
    // Manual operations
    public async Task<WalletDto> GetByTenantIdAsync(string tenantId)
    {
        var wallet = await _walletRepository.GetByTenantIdAsync(tenantId);
        return ObjectMapper.Map<Wallet, WalletDto>(wallet);
    }
    
    public async Task AddAssetAsync(AddAssetInput input)
    {
        var wallet = await _walletRepository.GetAsync(input.WalletId);
        wallet.AddAsset(input.AssetType, input.InitialBalance, input.ExpiresAt);
        await _walletRepository.UpdateAsync(wallet);
    }
}
```

---

## Acceptance Criteria

- [x] Wallet automatically created on tenant creation
- [x] One wallet per tenant enforced
- [x] Multi-asset support implemented
- [x] Balance operations respect business rules
- [x] Domain events published correctly
- [x] Tenant isolation enforced
- [x] Audit trail maintained
- [x] All business rules validated

---

## Related Documents

- [Wallet Domain](../../domains/wallet/) - Wallet aggregate documentation
- [Wallet Business Rules](../../domains/wallet/business-rules.md) - All wallet business rules
- [Wallet API](../../api/) - Wallet API endpoints
- [Wallet Events](../../domains/wallet/domain-events.md) - Published events

---

**Implementation Status**: ✅ **COMPLETE**  
**Last Updated**: 2026-07-27  
**Next Enhancement**: Add wallet analytics and reporting
