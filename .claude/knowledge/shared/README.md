# Shared Knowledge Base

**Purpose**: Common conventions, standards, and patterns shared across all WION microservices.

**Scope**: This documentation applies to **ALL services** in the WION ecosystem. Never duplicate these concepts in service-specific documentation.

---

## Overview

The **Shared Knowledge Base** contains conventions and standards that are:
- ✅ Universal across all microservices
- ✅ Maintained in one location (Single Source of Truth)
- ✅ Referenced by service-specific documentation
- ✅ Enforced consistently across services
- ✅ Aligned with Wion Engineering Rules

**Do NOT duplicate** shared concepts in service documentation. Instead, reference these shared documents.

---

## Wion Architecture Overview

### Project Structure

```
src/
├── framework/
│   └── Wion.BuildingBlock/          # ✅ Shared foundation (MUST USE THIS)
│       ├── Wion.BuildingBlock.Domain/
│       ├── Wion.BuildingBlock.Application/
│       └── Wion.BuildingBlock.Infrastructure/
├── services/
│   ├── billing-management/          # Existing TMT-style service
│   ├── wion-billing/                # New DDD-style service
│   └── [other services]/             # Future services
└── building-blocks/                  # ❌ DEPRECATED - Do not use
```

### Architecture Patterns

**Two Architecture Coexist**:

1. **TMT Pattern** (billing-management):
   - `ITMTRepository<TEntity, TKey>`
   - `TMTEfCoreRepository<TDbContext, TEntity, TKey>`
   - Established pattern in existing services

2. **DDD Pattern** (wion-billing):
   - Domain-Driven Design
   - Clean Architecture
   - CQRS where appropriate
   - Event-Driven Architecture

**Both patterns are valid**. Follow the existing pattern in each service unless migrating to a new pattern.

---

## Shared Documentation

### Engineering Standards

- **[Wion Engineering Rules](../../CLAUDE.md)** ⭐ **START HERE**
  - Mandatory rules for all tasks
  - Source of truth for engineering decisions
  - Definition of Done checklist

- **[DDD Conventions](./ddd-conventions.md)**
  - Domain-Driven Design patterns and principles
  - Aggregate design rules
  - Business rule conventions
  - Domain event patterns

- **[Event Conventions](./event-conventions.md)**
  - Event naming conventions
  - Event payload structure
  - Event versioning strategy
  - Cross-service event patterns

- **[API Conventions](./api-conventions.md)**
  - REST API patterns
  - Authentication standards
  - Error code conventions
  - Idempotency requirements

- **[Documentation Standards](./documentation-standards.md)**
  - Documentation structure patterns
  - Quality standards
  - Maintenance workflows
  - Template usage

- **[Glossary](./glossary.md)**
  - Universal DDD terminology
  - Clean Architecture definitions
  - Common acronyms
  - Shared concepts

---

## Quick Reference

### "Where do I find X?"

| Concept | Location | Applies To |
|---------|----------|-----------|
| **Engineering Rules** | `CLAUDE.md` | All tasks |
| **DDD patterns** | `shared/ddd-conventions.md` | DDD services |
| **TMT patterns** | `shared/repository-architecture.md` | TMT services |
| **Event patterns** | `shared/event-conventions.md` | All services |
| **API patterns** | `shared/api-conventions.md` | All services |
| **Service-specific** | `knowledge/{service}/` | One service |

---

## Usage Guidelines

### For Service Documentation

**DO**: Reference shared conventions
```markdown
## Domain Events

This service follows [WION event conventions](../shared/event-conventions.md).

### WalletCreated
Follows standard pattern: `{Aggregate}{StateChange}`
```

**DON'T**: Duplicate shared conventions
```markdown
## Domain Events

Event naming: Use past tense...  ❌ WRONG (duplicates shared docs)
```

---

### For AI Agents

**When working on a service**:
1. Load Wion Engineering Rules first (CLAUDE.md)
2. Load shared conventions
3. Load service-specific documentation
4. Apply shared patterns to service context

**Example workflow**:
```bash
# Step 1: Load engineering rules
load(CLAUDE.md)

# Step 2: Load shared conventions
load(knowledge/shared/ddd-conventions.md)
load(knowledge/shared/api-conventions.md)

# Step 3: Load service docs
load(knowledge/{service}/README.md)
load(knowledge/{service}/domains/)

# Step 4: Apply conventions
"This service follows event naming: {Aggregate}{StateChange}"
```

---

## Architecture Patterns Reference

### Repository Pattern Selection

**TMT Pattern** (Use in billing-management and existing TMT services):
```csharp
// Interface
public interface IWalletRepository : ITMTRepository<Wallet, long>
{
    Task<Wallet> FindByTenantIdAsync(string tenantId);
}

// Implementation
public class EfCoreWalletRepository : TMTEfCoreRepository<BillingDbContext, Wallet, long>, IWalletRepository
{
    public EfCoreWalletRepository(IDbContextProvider<BillingDbContext> dbContextProvider, IUniqueKey uniqueKey)
        : base(dbContextProvider, uniqueKey)
    {
    }
}
```

**DDD Pattern** (Use in wion-billing and new DDD services):
```csharp
// Interface
public interface IWalletRepository : IRepository<Wallet>
{
    Task<Wallet> FindByTenantIdAsync(string tenantId);
}

// Implementation
public class EfCoreWalletRepository : EfCoreRepository<BillingDbContext, Wallet>, IWalletRepository
{
    public EfCoreWalletRepository(IDbContextProvider<BillingDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }
}
```

**Key Decision**: Follow the existing pattern in each service. Don't mix patterns within the same service.

---

### Aggregate Pattern Selection

**TMT Style** (billing-management):
- Uses TMT-specific base classes
- Follows TMT naming conventions
- Integrates with TMT framework

**DDD Style** (wion-billing):
```csharp
public class Wallet : BillingAggregateRoot, ITMTMultiTenant
{
    // Private setters for domain state
    public decimal AvailableBalance { get; private set; }
    
    // Static factory method
    public static Wallet CreateNew(long id, string tenantId, string currency = "VND")
    {
        return new Wallet(id, tenantId, currency);
    }
    
    // Domain operations with business rules
    public void Credit(decimal amount)
    {
        CheckRule(new CreditAmountMustBePositiveRule(amount));
        CheckRule(new WalletNotDeletedRule(this));
        AvailableBalance += amount;
        AddLocalEvent(new BalanceChangedDomainEvent(...));
    }
}
```

---

## Maintaining Shared Documentation

### When to Update Shared Docs

**Update shared documentation when**:
- ✅ New universal pattern discovered
- ✅ Existing convention needs clarification
- ✅ Cross-service requirement emerges
- ✅ Architectural decision affects all services
- ✅ Wion Engineering Rules change

**Do NOT update shared docs when**:
- ❌ Only one service affected
- ❌ Service-specific optimization
- ❌ Temporary workaround

---

### Update Process

1. **Propose change**: Document rationale and impact
2. **Review across services**: Check for breaking changes
3. **Update shared doc**: Modify in this directory
4. **Notify services**: Update service docs to reference new convention
5. **Validate**: Ensure no duplication
6. **Test**: Verify build succeeds across affected services

---

## Conventions by Category

### Wion Engineering Rules

**Location**: `CLAUDE.md`

**Key rules**:
- Think Before Coding
- Source of Truth
- Wion.BuildingBlock usage
- Architecture principles
- Refactoring rules
- Definition of Done

**Used by**: All tasks, all services

---

### Domain-Driven Design

**Location**: `ddd-conventions.md`

**Key conventions**:
- Aggregate design patterns
- Business rule format
- Domain event naming
- Repository patterns
- Policy pattern

**Used by**: DDD-style services (wion-billing, new services)

---

### TMT Framework

**Location**: `repository-architecture.md`

**Key conventions**:
- TMT repository pattern
- TMT entity framework integration
- TMT-specific base classes
- TMT naming conventions

**Used by**: TMT-style services (billing-management, existing services)

---

### Events

**Location**: `event-conventions.md`

**Key conventions**:
- Event naming: `{Aggregate}{StateChange}`
- Event payload structure
- Event versioning
- Delivery guarantees
- Idempotency requirements

**Used by**: All services publishing/subscribing events

---

### APIs

**Location**: `api-conventions.md`

**Key conventions**:
- REST API patterns
- Error code format
- Authentication standards
- Idempotency keys
- Pagination format

**Used by**: All services exposing APIs

---

## Versioning

### Shared Documentation Versioning

**Current version**: 2.0

**Versioning rules**:
- Major version: Breaking changes
- Minor version: Additions, compatible changes
- Patch version: Fixes, clarifications

**Example**:
```
1.0 → 1.1  (Minor: Added new pattern)
1.1 → 2.0  (Major: Breaking change - Wion Engineering Rules added)
2.0 → 2.0.1  (Patch: Fixed typo)
```

---

## Related Documentation

### Service-Specific Documentation

Each service has its own Knowledge Base:
```
knowledge/
├── shared/              # This directory
├── wion-billing/        # WION Billing Service docs (DDD pattern)
├── billing-management/   # TMT Billing Service docs (TMT pattern)
└── [other services]/    # Future services
```

**Service-specific docs** should:
- ✅ Reference shared conventions
- ✅ Document service-specific patterns
- ✅ Follow Wion Engineering Rules
- ❌ NOT duplicate shared knowledge

---

## Important Architectural Decisions

### Decision: Support Both TMT and DDD Patterns

**Rationale**: 
- Existing services use TMT pattern
- New services can use DDD pattern
- Migration cost is high
- Both patterns are valid

**Guidance**:
- Follow existing pattern in each service
- Don't mix patterns within a service
- When creating new service, choose appropriate pattern
- Document pattern choice in service README

---

### Decision: Wion.BuildingBlock Location

**Correct Location**: `src/framework/Wion.BuildingBlock`

**Deprecated**: `src/building-blocks/`

**Rationale**: 
- Framework-level code belongs in framework directory
- Clearer separation from business services
- Better dependency management

**Migration**: All new shared code MUST go in `src/framework/Wion.BuildingBlock`

---

## Support

**Questions about shared conventions**?
- **Architecture Team**: architecture@wion.vn
- **Tech Lead**: tech-lead@wion.vn

**Questions about Wion Engineering Rules**?
- See main [CLAUDE.md](../../CLAUDE.md) for complete rules

---

## Index

### Quick Links

- [Wion Engineering Rules](../../CLAUDE.md) ⭐ **START HERE**
- [DDD Conventions](./ddd-conventions.md)
- [Event Conventions](./event-conventions.md)
- [API Conventions](./api-conventions.md)
- [Repository Architecture](./repository-architecture.md)
- [Glossary](./glossary.md)

---

**Last Updated**: 2026-07-24  
**Maintained By**: Architecture Team  
**Version**: 2.0 (Wion Engineering Rules + Multi-Pattern Support)
