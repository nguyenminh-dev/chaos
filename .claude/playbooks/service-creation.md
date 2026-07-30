# Service Creation Playbook

**Purpose**: Step-by-step guide for creating new WION microservices following established patterns and conventions.

---

## Prerequisites

### Required Knowledge
- Domain-Driven Design (DDD) principles
- ABP Framework patterns
- Clean Architecture concepts
- Repository testing standards
- WION platform conventions

### Required Tools
- .NET 6.0+ SDK
- Visual Studio 2022 or VS Code
- PostgreSQL client tools
- Git
- Docker (for local development)

---

## Service Creation Process

### Phase 1: Planning & Analysis

#### Step 1: Define Bounded Context
**Questions to answer**:
1. What is the service's primary responsibility?
2. What are the core business domains?
3. What are the service boundaries?
4. Who are the upstream/downstream services?

**Output**: [Service Bounded Context Document](../knowledge/shared/service-template.md)

#### Step 2: Identify Aggregates
**Questions to answer**:
1. What are the core business entities?
2. What are the aggregate roots?
3. What are the value objects?
4. What are the business rules?

**Output**: [Aggregate Design Document](../knowledge/shared/aggregate-template.md)

#### Step 3: Define APIs
**Questions to answer**:
1. What APIs will the service expose?
2. What external services will it integrate with?
3. What events will it publish/subscribe to?
4. What are the data models?

**Output**: [API Contract Document](../knowledge/shared/api-template.md)

---

### Phase 2: Service Scaffolding

#### Step 4: Create Solution Structure

**Using ABP CLI**:
```bash
# Install ABP CLI if not already installed
dotnet tool install -g Volo.Abp.Cli

# Create new ABP solution
abp new Wion.{ServiceName} -t app -- layered --ui none --database-provider postgresql
```

**Manual Setup** (if not using CLI):
1. Create solution structure following [ABP Conventions](../knowledge/shared/abp-conventions.md)
2. Set up project references
3. Configure dependencies
4. Set up module dependencies

---

#### Step 5: Configure Framework Modules

**Update Application Module**:
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
}
```

**Configure HTTP API Module**:
```csharp
[DependsOn(
    typeof({Service}ApplicationModule),
    typeof(AbpAspNetCoreMvcModule)
)]
public class {Service}HttpApiModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        ConfigureAuthorization();
    }
}
```

---

### Phase 3: Domain Implementation

#### Step 6: Implement Aggregates

**Create Aggregate Structure**:
```
Domain/{Aggregate}s/
├── {Aggregate}.cs                  # Aggregate Root
├── Events/                         # Domain Events
├── ValueObjects/                   # Value Objects
├── Rules/                         # Business Rules
├── I{Aggregate}Repository.cs      # Repository Interface
└── {Aggregate}Manager.cs          # Domain Service (optional)
```

**Implement Aggregate Root**:
```csharp
public class {Aggregate} : AggregateRootBase<Guid>, IAggregateRoot, ITMTMultiTenant
{
    public string TenantId { get; set; }
    public decimal MainProperty { get; private set; }
    
    private {Aggregate}() { }
    
    private {Aggregate}(Guid id, string tenantId)
    {
        Id = id;
        TenantId = tenantId;
        AddLocalEvent(new {Aggregate}CreatedDomainEvent(Id, tenantId));
    }
    
    internal static {Aggregate} CreateNew(string tenantId)
    {
        return new {Aggregate}(Guid.NewGuid(), tenantId);
    }
}
```

---

#### Step 7: Implement Business Rules

**Create Business Rule**:
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
        return _value < 0;
    }
    
    public string MessageCode => "Service:{BusinessRule}";
}
```

---

#### Step 8: Implement Repository Interface

**Define Repository**:
```csharp
public interface I{Aggregate}Repository : IRepository<{Aggregate}, Guid>
{
    Task<{Aggregate}> FindByTenantIdAsync(string tenantId);
    Task<bool> ExistsByTenantIdAsync(string tenantId);
}
```

---

### Phase 4: Application Implementation

#### Step 9: Create Application Contracts

**Define DTOs**:
```csharp
public class {Aggregate}Dto : EntityDto<Guid>
{
    public string TenantId { get; set; }
    public decimal Value { get; set; }
}

public class Create{Aggregate}Dto
{
    public string TenantId { get; set; }
    public decimal InitialValue { get; set; }
}

public interface I{Aggregate}AppService : IApplicationService
{
    Task<{Aggregate}Dto> GetAsync(Guid id);
    Task<{Aggregate}Dto> CreateAsync(Create{Aggregate}Dto input);
}
```

---

#### Step 10: Implement Application Service

**Create Application Service**:
```csharp
[Authorize({Service}Permissions.GroupName)]
public class {Aggregate}AppService : ApplicationService, I{Aggregate}AppService
{
    private readonly I{Aggregate}Repository _aggregateRepository;
    
    public {Aggregate}AppService(I{Aggregate}Repository aggregateRepository)
    {
        _aggregateRepository = aggregateRepository;
    }
    
    public async Task<{Aggregate}Dto> GetAsync(Guid id)
    {
        var aggregate = await _aggregateRepository.GetAsync(id);
        return ObjectMapper.Map<{Aggregate}, {Aggregate}Dto>(aggregate);
    }
    
    public async Task<{Aggregate}Dto> CreateAsync(Create{Aggregate}Dto input)
    {
        var aggregate = {Aggregate}.CreateNew(input.TenantId);
        await _aggregateRepository.InsertAsync(aggregate);
        return ObjectMapper.Map<{Aggregate}, {Aggregate}Dto>(aggregate);
    }
}
```

---

### Phase 5: Infrastructure Implementation

#### Step 11: Implement Repository

**Create EF Core Repository**:
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

#### Step 12: Configure Database Context

**Update DbContext**:
```csharp
public class {Service}DbContext : AbpDbContext
{
    public DbSet<{Aggregate}> {Aggregate}s { get; set; }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.Configure{Service}();
    }
}
```

---

### Phase 6: API Implementation

#### Step 13: Create API Controllers

**Create Controller**:
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
        return await _aggregateAppService.GetAsync(id);
    }
    
    [HttpPost]
    public async Task<{Aggregate}Dto> CreateAsync(Create{Aggregate}Dto input)
    {
        return await _aggregateAppService.CreateAsync(input);
    }
}
```

---

### Phase 7: Testing Implementation

#### Step 14: Implement Domain Tests

**Create Domain Tests**:
```csharp
public class {Aggregate}Tests : {Service}DomainTestBase
{
    [Fact]
    public void Should_Create_New_{Aggregate}()
    {
        // Arrange & Act
        var aggregate = {Aggregate}.CreateNew("tenant-123");
        
        // Assert
        aggregate.ShouldNotBeNull();
        aggregate.TenantId.ShouldBe("tenant-123");
    }
    
    [Fact]
    public void Should_Publish_{Aggregate}_Created_Event()
    {
        // Arrange & Act
        var aggregate = {Aggregate}.CreateNew("tenant-123");
        
        // Assert
        aggregate.GetLocalEvents()
            .Any(e => e is {Aggregate}CreatedDomainEvent)
            .ShouldBeTrue();
    }
}
```

---

#### Step 15: Implement Application Tests

**Create Application Tests**:
```csharp
public class {Aggregate}AppServiceTests : {Service}ApplicationTestBase
{
    private readonly I{Aggregate}AppService _aggregateAppService;
    
    public {Aggregate}AppServiceTests()
    {
        _aggregateAppService = GetRequiredService<I{Aggregate}AppService>();
    }
    
    [Fact]
    public async Task Should_Create_{Aggregate}()
    {
        // Arrange
        var input = new Create{Aggregate}Dto { TenantId = "tenant-123" };
        
        // Act
        var result = await _aggregateAppService.CreateAsync(input);
        
        // Assert
        result.ShouldNotBeNull();
        result.TenantId.ShouldBe("tenant-123");
    }
}
```

---

#### Step 16: Implement Integration Tests

**Create Integration Tests**:
```csharp
public class {Aggregate}RepositoryTests : {Service}IntegrationTestBase
{
    private readonly I{Aggregate}Repository _aggregateRepository;
    
    public {Aggregate}RepositoryTests()
    {
        _aggregateRepository = GetRequiredService<I{Aggregate}Repository>();
    }
    
    [Fact]
    public async Task Should_Insert_{Aggregate}_To_Database()
    {
        // Arrange
        var aggregate = {Aggregate}.CreateNew("tenant-123");
        
        // Act
        await _aggregateRepository.InsertAsync(aggregate);
        
        // Assert
        var found = await _aggregateRepository.FindAsync(aggregate.Id);
        found.ShouldNotBeNull();
    }
}
```

---

### Phase 8: Documentation

#### Step 17: Create Knowledge Base Documentation

**Create Service Documentation Structure**:
```
.claude/knowledge/{service}/
├── README.md                        # Service overview
├── architecture/
│   ├── bounded-context.md
│   ├── context-map.md
│   └── dependency-map.md
├── domains/
│   └── {domain}/
│       ├── overview.md
│       ├── aggregate.md
│       ├── business-rules.md
│       ├── lifecycle.md
│       └── domain-events.md
├── application/
│   └── use-cases/
├── api/
│   └── index.md
├── events/
│   └── README.md
└── reference/
    └── glossary.md
```

---

#### Step 18: Document Business Rules

**Create Business Rules Document**:
```markdown
# {Domain} Business Rules

## Core Business Rules

### BR-{D}-001: {Rule Name}
**Rule**: {Rule description}

**Formal Definition**: {Mathematical/logical definition}

**Enforcement Points**: {Where/how enforced}

**Violation Handling**: {What happens on violation}
```

---

#### Step 19: Document API Contracts

**Create API Documentation**:
```markdown
# {Service} API Documentation

## {Aggregate} Endpoints

### GET /api/{aggregate}/{id}
**Description**: Get {aggregate} by ID

**Request**:
- `id` (path): GUID identifier

**Response**:
```json
{
  "id": "guid",
  "tenantId": "string",
  "value": 0
}
```
```

---

### Phase 9: Validation & Review

#### Step 20: Self-Review Checklist

**Code Quality**:
- [ ] Follows DDD principles
- [ ] Follows ABP conventions
- [ ] Business rules in domain layer
- [ ] No code duplication
- [ ] SOLID principles respected

**Testing Coverage**:
- [ ] Domain tests pass
- [ ] Application tests pass
- [ ] Integration tests pass
- [ ] Critical paths covered
- [ ] Edge cases tested

**Documentation**:
- [ ] Business rules documented
- [ ] API contracts documented
- [ ] Architecture documented
- [ ] Knowledge base updated
- [ ] README files updated

**Architecture**:
- [ ] Aggregate boundaries respected
- [ ] Repository pattern followed
- [ ] Event handling implemented
- [ ] Error handling implemented
- [ ] Logging implemented

---

#### Step 21: Peer Review

**Review Checklist**:
- [ ] Architecture review completed
- [ ] Code review completed
- [ ] Testing review completed
- [ ] Documentation review completed
- [ ] Security review completed (if applicable)

---

### Phase 10: Deployment Preparation

#### Step 22: Configuration Setup

**Create Configuration Files**:
```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database={Service};User Id=postgres;Password=password;"
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

#### Step 23: Migration Setup

**Create Database Migration**:
```bash
dotnet ef migrations add Initial_{Service}_Schema
dotnet ef database update
```

---

#### Step 24: Build & Deployment

**Build Service**:
```bash
dotnet build Wion.{Service}.sln --configuration Release
```

**Run Tests**:
```bash
dotnet test Wion.{Service}.sln --configuration Release
```

**Package for Deployment**:
```bash
docker build -t wion-{service}:latest .
```

---

## Service Creation Checklist

### Planning Phase
- [ ] Bounded context defined
- [ ] Aggregates identified
- [ ] Business rules documented
- [ ] APIs defined
- [ ] Events identified

### Implementation Phase
- [ ] Solution structure created
- [ ] Domain layer implemented
- [ ] Application layer implemented
- [ ] Infrastructure layer implemented
- [ ] API layer implemented
- [ ] Tests implemented
- [ ] Documentation created

### Validation Phase
- [ ] All tests passing
- [ ] Code review completed
- [ ] Architecture review completed
- [ ] Security review completed
- [ ] Performance tested

### Deployment Phase
- [ ] Configuration setup
- [ ] Database migrations created
- [ ] Deployment pipeline configured
- [ ] Monitoring setup
- [ ] Documentation published

---

## Common Pitfalls to Avoid

### Architecture Pitfalls
- ❌ Starting implementation without bounded context analysis
- ❌ Placing business rules in application layer
- ❌ Violating aggregate boundaries
- ❌ Creating anemic domain models
- ❌ Ignoring domain events

### Testing Pitfalls
- ❌ Testing only happy paths
- ❌ Missing edge case testing
- ❌ Insufficient domain test coverage
- ❌ No integration testing
- ❌ Missing business rule tests

### Documentation Pitfalls
- ❌ Not documenting business rules
- ❌ Missing API documentation
- ❌ Outdated knowledge base
- ❌ No architecture documentation
- ❌ Missing event documentation

---

## Related Documentation
- [ABP Conventions](../knowledge/shared/abp-conventions.md) - Framework patterns
- [DDD Conventions](../knowledge/shared/ddd-conventions.md) - Domain design patterns
- [Testing Conventions](../knowledge/shared/testing-conventions.md) - Testing standards
- [Repository Architecture](../knowledge/shared/repository-architecture.md) - Repository patterns

---

## Support

**Questions about service creation**?
- **Architecture Team**: architecture@wion.vn
- **Tech Lead**: tech-lead@wion.vn

---

**Last Updated**: 2026-07-14  
**Maintained By**: Architecture Team  
**Version**: 1.0