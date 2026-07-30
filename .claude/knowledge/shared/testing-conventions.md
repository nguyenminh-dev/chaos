# Testing Conventions

**Purpose**: Standard testing patterns and conventions used across all WION microservices.

**Scope**: These conventions apply to **all services** in the WION ecosystem.

---

## Testing Philosophy

### Test Pyramid Approach

```
           E2E Tests (few)
          /             \
     Integration Tests (more)
    /                       \
Domain Tests & Unit Tests (most)
```

**Testing Strategy**:
- **Domain Tests** - Business rule validation
- **Application Tests** - Use case validation
- **Integration Tests** - Database and external service integration
- **E2E Tests** - Critical user workflows

---

## Testing Framework Stack

### .NET Testing Frameworks
- **xUnit** - Primary testing framework
- **Shouldly** - Assertion library
- **NSubstitute** - Mocking framework
- **Bogus** - Test data generation

### Test Organization
- **Domain.Tests** - Domain layer tests
- **Application.Tests** - Application layer tests
- **EntityFrameworkCore.Tests** - Infrastructure tests
- **HttpApi.Client.ConsoleTestApp** - API client tests
- **TestBase** - Shared test utilities

---

## Domain Layer Testing

### Test Structure Pattern

**Standard domain test organization**:
```
Domain.Tests/
├── {Aggregate}/
│   ├── {Aggregate}Tests.cs              # Aggregate behavior tests
│   ├── Rules/
│   │   └── BusinessRulesTests.cs        # Business rule tests
│   └── ValueObjects/
│       └── {ValueObject}Tests.cs        # Value object tests
```

---

### Aggregate Testing Pattern

**Standard aggregate test structure**:
```csharp
public class WalletTests : BillingDomainTestBase
{
    private readonly string _tenantId = "tenant-123";
    
    #region Creation Tests
    
    [Fact]
    public void Should_Create_New_Wallet()
    {
        // Arrange & Act
        var wallet = Wallet.CreateNew(_tenantId);
        
        // Assert
        wallet.Id.ShouldNotBe(Guid.Empty);
        wallet.TenantId.ShouldBe(_tenantId);
        wallet.AvailableBalance.ShouldBe(0);
        wallet.ReservedBalance.ShouldBe(0);
        wallet.Currency.ShouldBe("VND");
        wallet.Assets.Count.ShouldBe(0);
        wallet.IsDeleted.ShouldBeFalse();
    }
    
    #endregion
    
    #region Domain Event Tests
    
    [Fact]
    public void Should_Publish_Wallet_Created_Event_On_Creation()
    {
        // Arrange & Act
        var wallet = Wallet.CreateNew(_tenantId);
        
        // Assert
        var domainEvent = wallet.GetLocalEvents().FirstOrDefault();
        domainEvent.ShouldNotBeNull();
        domainEvent.ShouldBeOfType<WalletCreatedDomainEvent>();
        
        var walletEvent = (WalletCreatedDomainEvent)domainEvent;
        walletEvent.WalletId.ShouldBe(wallet.Id);
        walletEvent.TenantId.ShouldBe(_tenantId);
    }
    
    #endregion
    
    #region Business Operation Tests
    
    [Fact]
    public void Should_Credit_Wallet_Balance()
    {
        // Arrange
        var wallet = Wallet.CreateNew(_tenantId);
        
        // Act
        wallet.Credit(1000);
        
        // Assert
        wallet.AvailableBalance.ShouldBe(1000);
        wallet.ReservedBalance.ShouldBe(0);
        wallet.TotalBalance.ShouldBe(1000);
    }
    
    #endregion
    
    #region Business Rule Violation Tests
    
    [Fact]
    public void Should_Not_Allow_Credit_Amount_Less_Than_Or_Equal_To_Zero()
    {
        // Arrange
        var wallet = Wallet.CreateNew(_tenantId);
        
        // Act & Assert
        Should.Throw<BusinessRuleValidationException>(() => wallet.Credit(0));
        Should.Throw<BusinessRuleValidationException>(() => wallet.Credit(-100));
    }
    
    #endregion
    
    #region Complex Workflow Tests
    
    [Fact]
    public void Should_Handle_Complete_Balance_Operation_Flow()
    {
        // Arrange
        var wallet = Wallet.CreateNew(_tenantId);
        wallet.Credit(10000);
        
        // Act - Reserve 3000 for service
        wallet.Reserve(3000);
        wallet.AvailableBalance.ShouldBe(7000);
        wallet.ReservedBalance.ShouldBe(3000);
        
        // Act - Confirm 2000 deduction (service completed successfully)
        wallet.ConfirmDeduction(2000);
        wallet.AvailableBalance.ShouldBe(7000);
        wallet.ReservedBalance.ShouldBe(1000);
        
        // Act - Release remaining reserve (service partially completed)
        wallet.ReleaseReserve(1000);
        wallet.AvailableBalance.ShouldBe(8000);
        wallet.ReservedBalance.ShouldBe(0);
    }
    
    #endregion
}
```

---

### Business Rule Testing Pattern

**Standard business rule test**:
```csharp
public class BusinessRulesTests
{
    #region BalanceMustNotBeNegativeRule Tests
    
    [Fact]
    public void BalanceMustNotBeNegativeRule_Should_Pass_When_Balances_Are_Non_Negative()
    {
        // Arrange & Act
        var rule = new BalanceMustNotBeNegativeRule(1000, 200);
        
        // Assert
        rule.IsBroken().ShouldBeFalse();
    }
    
    [Fact]
    public void BalanceMustNotBeNegativeRule_Should_Fail_When_Available_Balance_Is_Negative()
    {
        // Arrange & Act
        var rule = new BalanceMustNotBeNegativeRule(-100, 200);
        
        // Assert
        rule.IsBroken().ShouldBeTrue();
        rule.MessageCode.ShouldBe("Billing:BalanceMustNotBeNegative");
    }
    
    #endregion
}
```

---

### Value Object Testing Pattern

**Standard value object test**:
```csharp
public class AssetTypeTests
{
    [Fact]
    public void Should_Create_Wi_Credit_Asset_Type()
    {
        // Arrange & Act
        var assetType = AssetType.WiCredit();
        
        // Assert
        assetType.Code.ShouldBe("WI_CREDIT");
        assetType.CanExpire.ShouldBeFalse();
    }
    
    [Fact]
    public void Should_Create_Promotion_Asset_Type_With_Expiration()
    {
        // Arrange & Act
        var assetType = AssetType.Promotion();
        
        // Assert
        assetType.Code.ShouldBe("PROMOTION");
        assetType.CanExpire.ShouldBeTrue();
    }
    
    [Fact]
    public void Asset_Types_Should_Be_Equal_With_Same_Code()
    {
        // Arrange
        var assetType1 = AssetType.WiCredit();
        var assetType2 = AssetType.WiCredit();
        
        // Act & Assert
        assetType1.Equals(assetType2).ShouldBeTrue();
    }
}
```

---

## Application Layer Testing

### Application Service Test Pattern

**Standard application service test**:
```csharp
public class WalletAppServiceTests : WalletApplicationTestBase
{
    private readonly IWalletAppService _walletAppService;
    
    public WalletAppServiceTests()
    {
        _walletAppService = GetRequiredService<IWalletAppService>();
    }
    
    [Fact]
    public async Task Should_Create_Wallet()
    {
        // Arrange
        var input = new CreateWalletDto
        {
            TenantId = "tenant-123",
            Currency = "VND"
        };
        
        // Act
        var result = await _walletAppService.CreateAsync(input);
        
        // Assert
        result.ShouldNotBeNull();
        result.TenantId.ShouldBe("tenant-123");
        result.AvailableBalance.ShouldBe(0);
    }
    
    [Fact]
    public async Task Should_Not_Create_Duplicate_Wallet()
    {
        // Arrange
        var input = new CreateWalletDto
        {
            TenantId = "tenant-123",
            Currency = "VND"
        };
        
        // Act
        await _walletAppService.CreateAsync(input);
        
        // Assert
        await Should.ThrowAsync<BusinessException>(
            async () => await _walletAppService.CreateAsync(input)
        );
    }
}
```

---

## Integration Testing

### Database Integration Test Pattern

**Standard integration test**:
```csharp
public class WalletRepositoryTests : WalletIntegrationTestBase
{
    private readonly IWalletRepository _walletRepository;
    
    public WalletRepositoryTests()
    {
        _walletRepository = GetRequiredService<IWalletRepository>();
    }
    
    [Fact]
    public async Task Should_Insert_Wallet_To_Database()
    {
        // Arrange
        var wallet = Wallet.CreateNew("tenant-123");
        
        // Act
        await _walletRepository.InsertAsync(wallet);
        
        // Assert
        var foundWallet = await _walletRepository.FindAsync(wallet.Id);
        foundWallet.ShouldNotBeNull();
        foundWallet.TenantId.ShouldBe("tenant-123");
    }
    
    [Fact]
    public async Task Should_Query_Wallet_By_Tenant_Id()
    {
        // Arrange
        var wallet = Wallet.CreateNew("tenant-123");
        await _walletRepository.InsertAsync(wallet);
        
        // Act
        var foundWallet = await _walletRepository.FindByTenantIdAsync("tenant-123");
        
        // Assert
        foundWallet.ShouldNotBeNull();
        foundWallet.Id.ShouldBe(wallet.Id);
    }
}
```

---

## Test Base Classes

### Domain Test Base Pattern

**Standard domain test base**:
```csharp
public abstract class BillingDomainTestBase
{
    protected BillingDomainTestBase()
    {
        // Test setup
    }
    
    protected IServiceProvider BuildServiceProvider()
    {
        var services = new ServiceCollection();
        
        services.AddLogging();
        services.AddDbContext<BillingDbContext>(options =>
        {
            options.UseInMemoryDatabase(Guid.NewGuid().ToString());
        });
        
        // Add domain services
        services.AddTransient<WalletManager>();
        
        return services.BuildServiceProvider();
    }
}
```

---

### Application Test Base Pattern

**Standard application test base**:
```csharp
public abstract class WalletApplicationTestBase : WalletTestBase<WalletApplicationTestModule>
{
    protected override void SetAbpApplicationCreationOptions(AbpApplicationCreationOptions options)
    {
        options.UseAutofac();
    }
    
    protected virtual Task WithRepositoryAsync(Func<IWalletRepository, Task> action)
    {
        return WithUnitOfWorkAsync(async () =>
        {
            var repository = GetRequiredService<IWalletRepository>();
            await action(repository);
        });
    }
}
```

---

## Test Data Management

### Test Data Generation Pattern

**Standard test data generation**:
```csharp
public class TestDataBuilder
{
    public static Wallet CreateValidWallet()
    {
        var wallet = Wallet.CreateNew("tenant-123");
        wallet.Credit(1000);
        return wallet;
    }
    
    public static Wallet CreateWalletWithAssets()
    {
        var wallet = Wallet.CreateNew("tenant-123");
        wallet.AddAsset(AssetType.WiCredit(), 1000, null);
        wallet.AddAsset(AssetType.Promotion(), 500, DateTime.UtcNow.AddDays(30));
        return wallet;
    }
    
    public static List<Wallet> CreateMultipleWallets(int count)
    {
        var wallets = new List<Wallet>();
        for (int i = 0; i < count; i++)
        {
            wallets.Add(Wallet.CreateNew($"tenant-{i}"));
        }
        return wallets;
    }
}
```

---

## Testing Best Practices

### Test Naming Conventions

**Standard test naming**:
```csharp
// Should_ExpectedBehavior_StateUnderTest
[Fact]
public void Should_Credit_Wallet_When_Balance_Is_Sufficient()

// Should_Not_Allow_InvalidOperation_StateUnderTest
[Fact]
public void Should_Not_Allow_Credit_When_Amount_Is_Negative()

// Should_Publish_EventName_When_ConditionMet
[Fact]
public void Should_Publish_Balance_Changed_Event_When_Credit_Occurs()
```

---

### Test Organization Principles

**Arrange-Act-Assert Pattern**:
```csharp
[Fact]
public void Should_Reserve_Balance_From_Wallet()
{
    // Arrange
    var wallet = Wallet.CreateNew(_tenantId);
    wallet.Credit(1000);
    
    // Act
    wallet.Reserve(300);
    
    // Assert
    wallet.AvailableBalance.ShouldBe(700);
    wallet.ReservedBalance.ShouldBe(300);
    wallet.TotalBalance.ShouldBe(1000);
}
```

---

### Test Isolation Principles

**Each test should be**:
- **Independent** - No dependencies on other tests
- **Repeatable** - Same results every time
- **Fast** - Quick execution
- **Clear** - Easy to understand

---

## Coverage Requirements

### Minimum Coverage Standards
- **Domain Layer**: 90%+ coverage
- **Application Layer**: 80%+ coverage
- **Infrastructure Layer**: 70%+ coverage
- **Critical Paths**: 100% coverage

### Coverage Focus Areas
- **Business Rules** - All business rules must be tested
- **Domain Events** - All events must be tested
- **Error Conditions** - All error paths must be tested
- **Edge Cases** - Boundary conditions must be tested

---

## Performance Testing

### Load Testing Patterns
- **Credit Consumption** - 10,000 TPS
- **Balance Queries** - < 100ms P95
- **Concurrent Operations** - Thread safety validation

---

## Related Documentation
- [ABP Conventions](./abp-conventions.md) - Framework testing patterns
- [DDD Conventions](./ddd-conventions.md) - Domain testing patterns
- [Repository Architecture](./repository-architecture.md) - Testing infrastructure

---

**Last Updated**: 2026-07-14  
**Maintained By**: Architecture Team  
**Version**: 1.0