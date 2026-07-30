# ABP Framework Conventions

**Purpose**: Standard ABP (ASP.NET Boilerplate) framework implementation patterns used across all WION microservices.

**Scope**: These conventions apply to **all ABP-based services** in the WION ecosystem.

---

## Solution Architecture

### Standard Layer Structure

All services follow the **ABP layered architecture**:

```
{Service}.sln
├── src/
│   ├── {Service}.Domain                   # Domain layer
│   ├── {Service}.Domain.Shared            # Shared domain concepts
│   ├── {Service}.Application              # Application layer
│   ├── {Service}.Application.Contracts    # Application contracts
│   ├── {Service}.EntityFrameworkCore      # Infrastructure implementation
│   ├── {Service}.HttpApi                   # API layer
│   ├── {Service}.HttpApi.Client           # Client SDK
│   ├── {Service}.HttpApi.Host             # API host
│   └── {Service}.DbMigrator               # Database migration tool
└── test/
    ├── {Service}.Domain.Tests              # Domain tests
    ├── {Service}.Application.Tests         # Application tests
    ├── {Service}.EntityFrameworkCore.Tests # Infrastructure tests
    ├── {Service}.TestBase                  # Test base classes
    └── {Service}.HttpApi.Client.ConsoleTestApp # Client tests
```

---

## Domain Layer Conventions

### Aggregate Structure Pattern

**Standard aggregate organization**:
```
Domain/
├── {Aggregate}/
│   ├── {Aggregate}.cs                    # Aggregate Root
│   ├── {Entity}.cs                      # Related entities
│   ├── Events/                          # Domain events
│   │   ├── {Aggregate}CreatedDomainEvent.cs
│   │   └── {Aggregate}UpdatedDomainEvent.cs
│   ├── ValueObjects/                     # Value objects
│   │   └── {ValueObject}.cs
│   ├── Rules/                           # Business rules
│   │   └── {BusinessRule}Rule.cs
│   ├── I{Aggregate}Repository.cs        # Repository interface
│   └── {Aggregate}Manager.cs            # Domain service (if needed)
```

**Example**: `Wallet` aggregate structure
```
Domain/Wallets/
├── Wallet.cs                    # Aggregate Root
├── WalletAsset.cs              # Related entity
├── Events/
│   ├── WalletCreatedDomainEvent.cs
│   └── BalanceChangedDomainEvent.cs
├── ValueObjects/
│   ├── Balance.cs
│   ├── AssetType.cs
│   └── Currency.cs
├── Rules/
│   ├── BalanceMustNotBeNegativeRule.cs
│   ├── SufficientBalanceRule.cs
│   └── WalletNotDeletedRule.cs
├── IWalletRepository.cs
└── WalletManager.cs           # Optional domain service
```

---

### Aggregate Root Pattern

**Standard aggregate root implementation**:
```csharp
public class {Aggregate} : AggregateRootBase<Guid>, IAggregateRoot, ITMTMultiTenant
{
    // Properties with private setters
    public string TenantId { get; set; }
    public decimal MainProperty { get; private set; }
    public DateTime? DeletedAt { get; private set; }
    
    // Collections
    public List<{RelatedEntity}> RelatedEntities { get; private set; }
    
    // Computed properties
    public bool IsDeleted => DeletedAt.HasValue;
    public decimal TotalValue => MainProperty + RelatedEntities.Sum(x => x.Value);
    
    // Private constructor for ORM
    private {Aggregate}()
    {
        RelatedEntities = new List<{RelatedEntity}>();
    }
    
    // Domain constructor with factory method pattern
    private {Aggregate}(Guid id, string tenantId)
    {
        Id = id;
        TenantId = tenantId;
        // Initialize properties
        MainProperty = 0;
        RelatedEntities = new List<{RelatedEntity}();
        
        // Publish domain event
        AddLocalEvent(new {Aggregate}CreatedDomainEvent(Id, tenantId));
    }
    
    // Factory method
    internal static {Aggregate} CreateNew(string tenantId)
    {
        return new {Aggregate}(Guid.NewGuid(), tenantId);
    }
    
    // Business methods with rule validation
    internal void PerformBusinessOperation(decimal amount)
    {
        CheckRule(new BusinessRuleValidationException(this, amount));
        
        MainProperty += amount;
        
        AddLocalEvent(new BusinessOperationOccurredDomainEvent(Id, amount));
    }
}
```

---

### Business Rule Pattern

**Standard business rule implementation**:
```csharp
public class {BusinessRule}Rule : IBusinessRule
{
    private readonly {Type} _value;
    
    public {BusinessRule}Rule({Type} value)
    {
        _value = value;
    }
    
    public bool IsBroken()
    {
        // Rule validation logic
        return _value < 0;
    }
    
    public string MessageCode => "Service:{BusinessRule}";
    
    public string Message => $"Business rule violated: {_value}";
}
```

**Usage in domain**:
```csharp
internal void PerformOperation(decimal amount)
{
    CheckRule(new {BusinessRule}Rule(amount));
    // Business logic
}
```

---

## Application Layer Conventions

### Application Service Pattern

**Standard application service**:
```csharp
[Authorize({Service}Permissions.GroupName)]
public class {Aggregate}AppService : ApplicationService, I{Aggregate}AppService
{
    private readonly I{Aggregate}Repository _aggregateRepository;
    
    public {Aggregate}AppService(I{Aggregate}Repository aggregateRepository)
    {
        _aggregateRepository = aggregateRepository;
    }
    
    public async Task<{Aggregate}Dto> GetAsync(Get{Aggregate}Dto input)
    {
        var aggregate = await _aggregateRepository.FindAsync(input.Id);
        if (aggregate == null)
        {
            throw new BusinessException({Service}DomainErrorCodes.{Aggregate}NotFound);
        }
        
        return ObjectMapper.Map<{Aggregate}, {Aggregate}Dto>(aggregate);
    }
    
    public async Task<{Aggregate}Dto> CreateAsync(Create{Aggregate}Dto input)
    {
        // Validate business preconditions
        if (await _aggregateRepository.ExistsAsync(input.TenantId))
        {
            throw new BusinessException({Service}DomainErrorCodes.{Aggregate}AlreadyExists);
        }
        
        // Create aggregate
        var aggregate = {Aggregate}.CreateNew(input.TenantId);
        
        // Persist
        await _aggregateRepository.InsertAsync(aggregate);
        
        return ObjectMapper.Map<{Aggregate}, {Aggregate}Dto>(aggregate);
    }
}
```

---

### DTO Organization Pattern

**Application.Contracts structure**:
```
Application.Contracts/
├── {Aggregate}/
│   ├── I{Aggregate}AppService.cs          # Service interface
│   ├── {Aggregate}Dto.cs                  # Read DTO
│   ├── Create{Aggregate}Dto.cs            # Create DTO
│   ├── Update{Aggregate}Dto.cs            # Update DTO
│   ├── Get{Aggregate}Dto.cs               # Query DTO
│   └── {Operation}Dto.cs                  # Operation-specific DTOs
```

**DTO conventions**:
- Use `concrete class` for all DTOs
- Inherit from `EntityDto<Guid>` for entity DTOs
- Include validation attributes
- Use `AutoMap` attributes for mapping

---

### Application Module Configuration

**Standard application module**:
```csharp
[DependsOn(
    typeof({Service}DomainModule),
    typeof(AbpAutoMapperModule)
)]
public class {Service}ApplicationModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        ConfigureAutoMapper();
    }
    
    private void ConfigureAutoMapper()
    {
        Configure<AbpAutoMapperOptions>(options =>
        {
            options.AddMaps<{Service}ApplicationModule>();
        });
    }
}
```

---

## API Layer Conventions

### Controller Pattern

**Standard API controller**:
```csharp
[Route("api/{aggregate}")]
public class {Aggregate}Controller : AbpController
{
    private readonly I{Aggregate}AppService _aggregateAppService;
    
    public {Aggregate}Controller(I{Aggregate}AppService aggregateAppService)
    {
        _aggregateAppService = aggregateAppService;
    }
    
    [HttpGet("{id}")]
    public async Task<{Aggregate}Dto> GetAsync(Guid id)
    {
        return await _aggregateAppService.GetAsync(new Get{Aggregate}Dto { Id = id });
    }
    
    [HttpPost]
    public async Task<{Aggregate}Dto> CreateAsync(Create{Aggregate}Dto input)
    {
        return await _aggregateAppService.CreateAsync(input);
    }
}
```

---

### HTTP API Module Pattern

**Standard HTTP API module**:
```csharp
[DependsOn(
    typeof({Service}ApplicationModule),
    typeof({Service}HttpApiModule)
)]
public class {Service}HttpApiModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        ConfigureAuthentication();
        ConfigureAuthorization();
    }
    
    private void ConfigureAuthorization()
    {
        Configure<AbpAuthorizationOptions>(options =>
        {
            options.AuthorizationPolicyProviders.Add<{Service}AuthorizationPolicyProvider>();
        });
    }
}
```

---

## Infrastructure Layer Conventions

### Repository Implementation Pattern

**Standard repository pattern**:
```csharp
public class EfCore{Aggregate}Repository : EfCoreRepository<{Service}DbContext, {Aggregate}, Guid>, I{Aggregate}Repository
{
    public EfCore{Aggregate}Repository(IDbContextProvider<{Service}DbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }
    
    public async Task<{Aggregate}> FindByTenantIdAsync(string tenantId)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet.FirstOrDefaultAsync(x => x.TenantId == tenantId);
    }
    
    public async Task<bool> ExistsByTenantIdAsync(string tenantId)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet.AnyAsync(x => x.TenantId == tenantId);
    }
}
```

---

### DbContext Configuration

**Standard DbContext setup**:
```csharp
public class {Service}DbContext : AbpDbContext
{
    public DbSet<{Aggregate}> {Aggregate}s { get; set; }
    
    public {Service}DbContext(DbContextOptions<{Service}DbContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        builder.Configure{Service}();
    }
}
```

---

## Testing Conventions

### Test Structure Pattern

**Standard test organization**:
```
Domain.Tests/
├── {Aggregate}/
│   ├── {Aggregate}Tests.cs               # Main aggregate tests
│   ├── Rules/
│   │   └── BusinessRulesTests.cs         # Business rule tests
│   └── ValueObjects/
│       └── {ValueObject}Tests.cs        # Value object tests
```

---

### Domain Test Pattern

**Standard domain test format**:
```csharp
public class {Aggregate}Tests : {Service}DomainTestBase
{
    [Fact]
    public async Task Should_Create_New_{Aggregate}()
    {
        // Arrange
        var tenantId = "tenant-123";
        
        // Act
        var aggregate = await {Aggregate}.CreateNew(tenantId);
        
        // Assert
        aggregate.Id.ShouldNotBe(Guid.Empty);
        aggregate.TenantId.ShouldBe(tenantId);
        aggregate.MainProperty.ShouldBe(defaultValue);
    }
    
    [Fact]
    public async Task Should_Publish_{Aggregate}_Created_Event()
    {
        // Arrange & Act
        var aggregate = await {Aggregate}.CreateNew("tenant-123");
        
        // Assert
        var domainEvent = aggregate.GetLocalEvents()
            .FirstOrDefault(e => e is {Aggregate}CreatedDomainEvent);
        
        domainEvent.ShouldNotBeNull();
        domainEvent.ShouldBeOfType<{Aggregate}CreatedDomainEvent>();
    }
}
```

---

### Business Rule Test Pattern

**Standard business rule test**:
```csharp
public class BusinessRulesTests
{
    [Fact]
    public void {BusinessRule}Rule_Should_Pass_When_Condition_Met()
    {
        // Arrange & Act
        var rule = new {BusinessRule}Rule(validValue);
        
        // Assert
        rule.IsBroken().ShouldBeFalse();
    }
    
    [Fact]
    public void {BusinessRule}Rule_Should_Fail_When_Condition_Not_Met()
    {
        // Arrange & Act
        var rule = new {BusinessRule}Rule(invalidValue);
        
        // Assert
        rule.IsBroken().ShouldBeTrue();
        rule.MessageCode.ShouldBe("Service:{BusinessRule}");
    }
}
```

---

## Module Dependencies

### Standard Dependency Pattern

**Module dependency hierarchy**:
```
{Service}.Domain.Shared (no dependencies)
    ↓
{Service}.Domain (→ Domain.Shared)
    ↓
{Service}.Application.Contracts (→ Domain.Shared)
    ↓
{Service}.Application (→ Domain, Application.Contracts)
    ↓
{Service}.EntityFrameworkCore (→ Domain)
    ↓
{Service}.HttpApi (→ Application.Contracts)
    ↓
{Service}.HttpApi.Host (→ All layers)
```

---

## Naming Conventions

### File Naming
- `{Aggregate}.cs` - Aggregate root
- `I{Aggregate}Repository.cs` - Repository interface
- `{Aggregate}AppService.cs` - Application service
- `{Aggregate}Controller.cs` - API controller
- `{Aggregate}Dto.cs` - Data transfer object
- `{Aggregate}Manager.cs` - Domain service

### Namespace Naming
- `Wion.{Service}.Domains.{Aggregate}s` - Domain layer
- `Wion.{Service}.Application.{Aggregate}s` - Application layer
- `Wion.{Service}.HttpApi.Controllers` - API layer

---

## Common ABP Patterns

### Permission Definition
```csharp
public class {Service}Permissions
{
    public const string GroupName = "{Service}";
    
    public static class {Aggregate}s
    {
        public const string Default = GroupName + ".{Aggregate}s";
        public const string Create = Default + ".Create";
        public const string Update = Default + ".Update";
        public const string Delete = Default + ".Delete";
    }
}
```

### Error Code Definition
```csharp
public class {Service}DomainErrorCodes
{
    public const string {Aggregate}NotFound = "{Service}:{Aggregate}NotFound";
    public const string {Aggregate}AlreadyExists = "{Service}:{Aggregate}AlreadyExists";
}
```

---

## Infrastructure Patterns

### DbContext Configuration Patterns

**Standard DbContext Implementation**:
```csharp
[ConnectionStringName("Default")]
public class {Service}DbContext : AbpDbContext<{Service}DbContext>, IHasEventOutbox, IHasEventInbox
{
    // Event tracking for distributed events
    public DbSet<OutgoingEventRecord> OutgoingEvents { get; set; }
    public DbSet<IncomingEventRecord> IncomingEvents { get; set; }
    
    // Business entities
    public DbSet<{Aggregate}> {Aggregate}s { get; set; }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Configure owned entities
        builder.Entity<{Aggregate}>(b =>
        {
            b.ToTable({Service}Consts.DbTablePrefix + "{Aggregate}s", {Service}Consts.DbSchema);
            b.ConfigureByConvention();
            
            // Configure properties
            b.Property(x => x.TenantId).IsRequired().HasMaxLength(256);
            b.Property(x => x.Name).IsRequired().HasMaxLength(256);
            
            // Configure indexes
            b.HasIndex(x => x.TenantId).IsUnique();
            
            // Configure owned entities
            b.OwnsMany(x => x.OwnedEntities, owned =>
            {
                owned.WithOwner().HasForeignKey("{Aggregate}Id");
                owned.Property("Property").IsRequired().HasMaxLength(50);
                owned.HasIndex("Property", "{Aggregate}Id").IsUnique();
            });
        });
    }
}
```

**Multi-Tenancy Filter Pattern**:
```csharp
protected override bool ShouldFilterEntity<TEntity>(IMutableEntityType entityType)
{
    if (typeof(ITMTMultiTenant).IsAssignableFrom(typeof(TEntity)))
    {
        return true;
    }
    return base.ShouldFilterEntity<TEntity>(entityType);
}

protected override Expression<Func<TEntity, bool>> CreateFilterExpression<TEntity>()
{
    var expression = base.CreateFilterExpression<TEntity>();
    
    if (typeof(ITMTMultiTenant).IsAssignableFrom(typeof(TEntity)))
    {
        Expression<Func<TEntity, bool>> multiTenantFilter = 
            e => !IsTMTTenantFilterEnabled || EF.Property<string>(e, "TenantId") == TMTCurrentTenant.Id;
        expression = expression == null ? multiTenantFilter : CombineExpressions(expression, multiTenantFilter);
    }
    return expression;
}
```

---

### Repository Implementation Patterns

**Standard Repository with Advanced Queries**:
```csharp
public class EfCore{Aggregate}Repository : TMTEfCoreRepository<{Service}DbContext, {Aggregate}, Guid>, I{Aggregate}Repository
{
    public EfCore{Aggregate}Repository(IDbContextProvider<{Service}DbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }
    
    public async Task<{Aggregate}> FindByTenantIdAsync(string tenantId)
    {
        var queryable = await GetQueryableAsync();
        return await queryable
            .Include(x => x.RelatedEntities)
            .Where(x => x.TenantId == tenantId && !x.IsDeleted)
            .FirstOrDefaultAsync();
    }
    
    public async Task<(long TotalCount, List<{Aggregate}> Items)> GetListPagedAndFilterAsync({Aggregate}GetListInput input)
    {
        var queryable = await GetQueryableAsync();
        
        // Filtering
        queryable = queryable
            .AsNoTracking()
            .WhereIf(input.AuthorId != null, x => x.AuthorId == input.AuthorId)
            .WhereIf(!input.SearchText.IsNullOrEmpty(), x => x.Name.Contains(input.SearchText));
        
        var totalCount = await queryable.LongCountAsync();
        
        // Sorting
        queryable = queryable
            .OrderByDescending(x => x.CreationTime)
            .ThenBy(x => x.Name);
        
        // Paging
        queryable = queryable
            .Skip(input.SkipCount)
            .Take(input.MaxResultCount);
        
        return (totalCount, await queryable.ToListAsync());
    }
}
```

---

## Integration Patterns

### gRPC Service Pattern

**Standard gRPC Service**:
```csharp
public class {Aggregate}GrpcService : {Aggregate}Grpc.{Aggregate}GrpcBase
{
    private readonly I{Aggregate}Repository _aggregateRepository;
    private readonly IStringLocalizer<{Service}Resource> _localizer;
    
    public {Aggregate}GrpcService(
        I{Aggregate}Repository aggregateRepository,
        IStringLocalizer<{Service}Resource> localizer)
    {
        _aggregateRepository = aggregateRepository;
        _localizer = localizer;
    }
    
    public override async Task<{Aggregate}GetReply> Get{Aggregate}Info({Aggregate}GetRequest request, ServerCallContext context)
    {
        var aggregate = await _aggregateRepository.FindAsync(request.Id);
        
        var reply = new {Aggregate}GetReply();
        
        if (aggregate == null)
        {
            reply.IsSuccess = false;
            reply.ErrorMessage = "{Aggregate} not found";
            return reply;
        }
        
        reply.IsSuccess = true;
        reply.Id = aggregate.Id;
        reply.Name = aggregate.Name;
        // Map other properties...
        
        return reply;
    }
}
```

---

### Event Handling Patterns

**Distributed Event Handler**:
```csharp
public class {Event}Handler : IDistributedEventHandler<{Event}Eto>, ITransientDependency
{
    private readonly I{Aggregate}Repository _aggregateRepository;
    private readonly ILogger<{Event}Handler> _logger;
    
    public {Event}Handler(
        I{Aggregate}Repository aggregateRepository,
        ILogger<{Event}Handler> logger)
    {
        _aggregateRepository = aggregateRepository;
        _logger = logger;
    }
    
    public async Task HandleEventAsync({Event}Eto eventData)
    {
        _logger.LogInformation("[{Service}] Processing event", eventData);
        
        // Handle event logic
        var aggregate = await _aggregateRepository.FindAsync(eventData.AggregateId);
        if (aggregate != null)
        {
            // Process event
            await _aggregateRepository.UpdateAsync(aggregate);
        }
    }
}
```

---

## Configuration Patterns

### Application Configuration

**Standard appsettings.json**:
```json
{
  "App": {},
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database={service};User ID=postgres;Password=password;"
  },
  "StringEncryption": {
    "DefaultPassPhrase": "encryption-key"
  },
  "Redis": {
    "ConnectionString": "localhost:6379"
  },
  "RabbitMQ": {
    "Connections": {
      "Default": "host=localhost"
    }
  }
}
```

---

## Module Dependency Patterns

**Infrastructure Module Configuration**:
```csharp
[DependsOn(
    typeof({Service}DomainModule),
    typeof(AbpEntityFrameworkCoreModule),
    typeof(TMTEntityFrameworkCoreModule)
)]
public class {Service}EntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Configure DbContext
        context.Services.AddAbpDbContext<{Service}DbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
        });
    }
}
```

---

## Related Documentation
- [DDD Conventions](./ddd-conventions.md) - Domain-driven design patterns
- [API Conventions](./api-conventions.md) - REST API patterns
- [Event Conventions](./event-conventions.md) - Event patterns
- [Testing Conventions](./testing-conventions.md) - Testing patterns
- [Repository Architecture](./repository-architecture.md) - Repository patterns

---

**Last Updated**: 2026-07-14  
**Maintained By**: Architecture Team  
**Version**: 1.1