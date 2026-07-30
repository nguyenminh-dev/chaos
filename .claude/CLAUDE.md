# Wion Engineering Rules

These rules are mandatory for every task in this repository.

Failure to follow these rules is considered an incorrect implementation.

---

# 1. Think Before Coding

For every feature, refactoring, migration or bug fix:

1. Analyze the existing codebase.
2. Understand the architecture.
3. Identify all affected projects.
4. Produce an implementation plan.
5. Review the plan against the architecture.
6. Execute the plan automatically.

Do **NOT** stop after creating the plan.

Do **NOT** ask for confirmation unless explicitly requested.

---

# 2. Source of Truth

The existing repository is always the source of truth.

Never invent a new architecture if one already exists.

Before creating any class, service or abstraction:

* search the repository
* reuse existing code
* extend existing implementations when appropriate

Never duplicate code.

---

# 3. Wion.BuildingBlock

Never create new shared code inside:

```
src/building-blocks
```

All reusable infrastructure must belong to:

```
src/framework/Wion.BuildingBlock
```

The BuildingBlock project is the only shared foundation for Wion services.

Only generic infrastructure belongs here.

Examples:

* Result
* Exceptions
* Guard
* Extensions
* BaseEntity
* AggregateRoot
* ValueObject
* Domain Events
* Event Bus abstraction
* Unit of Work abstraction
* Repository abstraction
* Specifications
* Validation
* Authorization abstraction
* Caching abstraction
* Distributed Lock
* Background Job abstraction
* Shared Middleware
* Shared Configuration
* Shared Contracts
* Shared DTO
* Common Constants
* Common Interfaces

Never move business logic into BuildingBlock.

---

# 4. Billing Rules

Billing is a bounded context.

Never place Billing business logic inside BuildingBlock.

Billing should consume BuildingBlock.

BuildingBlock must never depend on Billing.

---

# 5. Architecture

Always follow:

* Clean Architecture
* Test Driven Design - TDD
* DDD
* SOLID
* CQRS where appropriate
* Dependency Inversion

Never introduce circular dependencies.

Dependencies must always point inward.

---

# 6. Refactoring Rules

Before creating anything:

Search first.

If similar code exists:

* reuse it
* improve it
* extract it

Never create duplicate implementations.

Never copy & paste code.

---

# 7. Namespace Rules

Namespaces must match folder structure.

Avoid namespace inconsistencies.

Update all references after moving files.

---

# 8. Project References

Keep project references minimal.

Do not reference an application project from another application project.

Prefer abstractions.

Avoid unnecessary package references.

---

# 9. Dependency Injection

Every new service must be registered.

Avoid duplicate registrations.

Remove unused registrations.

---

# 10. Build Quality

A task is NOT complete until:

* solution builds
* no compile errors
* no warnings introduced unnecessarily
* no broken references
* no missing DI registrations

---

# 11. Migration Rules

When moving code:

* update namespaces
* update references
* remove old implementations
* remove dead code
* remove duplicate code

Never leave both old and new implementations.

---

# 12. Code Quality

Prefer:

* composition
* immutable objects
* small classes
* small methods
* descriptive names

Avoid:

* giant classes
* static state
* duplicated logic
* magic strings
* magic numbers

---

# 13. Testing

When changing shared code:

Run affected tests.

If tests fail:

Fix them.

Do not ignore failures.

---

# 14. Self Review

Before finishing any task, review:

* architecture
* dependencies
* naming
* performance
* maintainability
* code duplication
* dead code

Fix every issue found before considering the task complete.

---

# 15. Continuous Improvement

If a better reusable abstraction is discovered during implementation:

Extract it into:

```
src/framework/Wion.BuildingBlock
```

without waiting for explicit instructions.

---

# 16. Never Stop Halfway

Do not stop after:

* analysis
* planning
* generating code
* partial refactoring

Continue until:

* implementation is complete
* migration is complete
* build succeeds
* tests pass
* dead code is removed

---

# 17. Repository Awareness

Before modifying any module:

* inspect related services
* inspect shared libraries
* inspect existing abstractions
* understand dependency graph

Never implement in isolation.

---

# 18. Backward Compatibility

Avoid breaking public APIs.

If breaking changes are unavoidable:

* migrate all usages
* update every caller
* keep the repository compiling

Never leave partially migrated code.

---

# 19. Engineering Principle

Think like the principal engineer responsible for the entire repository.

Optimize for long-term maintainability rather than the fastest implementation.

Every change should make the repository cleaner than before.

---

# 20. Definition of Done

A task is finished only when all of the following are true:

- Analysis completed.
- Implementation plan completed.

## Test-First Development

- Unit tests were written BEFORE any production code.
- The initial test run failed for the expected reason.
- Production code was implemented only after the failing tests existed.
- All new behavior is covered by unit tests.
- All tests pass.

## Implementation

- Implementation completed.
- Refactoring completed.
- Migration completed when required.

## Quality

- Build successful.
- No compiler warnings introduced.
- Dead code removed.
- Duplicate code removed.
- Architecture remains clean.
- Documentation updated when necessary.

## Validation

Include a final TDD report showing:
- Tests created first.
- Initial test result: FAILED.
- Production code implemented.
- Final test result: PASSED.

---

# Project Structure

## Multi-Service Architecture

This repository contains multiple microservices following Wion engineering standards:

```
src/
├── framework/
│   └── Wion.BuildingBlock/          # Shared foundation (MUST USE THIS)
├── services/
│   ├── billing-management/          # Existing billing service (TMT pattern)
│   ├── wion-billing/                # New billing service (DDD pattern)
│   └── [other services]/             # Future services
└── building-blocks/                  # ❌ DEPRECATED - Do not use
```

## Service Discovery Rules

### Rule 1: Automatic Service Detection

Before starting any task, identify affected services:

**Detection Methods**:
- ✅ **Explicit mention**: Task mentions service name ("Fix billing bug")
- ✅ **File paths**: Task includes service-specific paths (`knowledge/wion-billing/`)
- ✅ **Domain context**: Task mentions domain concepts ("Wallet", "Payment")
- ✅ **API context**: Task mentions API endpoints (`/api/v1/wallets/`)
- ✅ **Event context**: Task mentions service events (`PaymentSucceeded`)

### Rule 2: Knowledge Base Location

**NEVER hardcode service names in paths**.

**Correct Structure**:
```
.claude/
├── CLAUDE.md              # This file
├── playbooks/             # Global AI workflows
├── templates/             # Global documentation templates
├── prompts/               # AI prompt templates
└── knowledge/             # Multi-service documentation
    ├── shared/            # Shared conventions
    ├── wion-billing/      # WION Billing Service docs
    ├── billing-management/ # TMT Billing Management docs
    └── [other services]/  # Future services
```

## Standard Workflow

### For Every Task

**Step 1: Service Discovery**
```bash
# Detect affected service(s)
services = detect_services(task_context)
```

**Step 2: Load Shared Conventions**
```bash
load(knowledge/shared/ddd-conventions.md)
load(knowledge/shared/api-conventions.md)
load(knowledge/shared/event-conventions.md)
```

**Step 3: Load Service Documentation**
```bash
for service in services:
    load(knowledge/{service}/README.md)
    load(relevant_domain_docs)
```

**Step 4: Apply Context**
```bash
apply_shared_conventions_to_service(service)
```

**Step 5: Execute Task**
```bash
perform_task_with_context(service, shared_conventions)
```

---

# DDD Implementation Rules

## Rule 1

Never start coding immediately.

Always understand the business first.

Identify:

- Business capability
- Use Case
- Aggregate
- Invariants
- Domain Events
- Integration Events
- Transaction Boundary

If these are unclear:

STOP

Ask questions.

Never guess.

---

## Rule 2

Think in Domain first.

Never think:

Controller

↓

Service

↓

Repository

Instead think:

Business

↓

Domain

↓

Use Case

↓

Application

↓

Infrastructure

↓

API

---

## Rule 3

Practice Outside-In TDD

For every feature:

**Step 1**: Understand requirements.

**Step 2**: Identify Acceptance Criteria.

**Step 3**: Design API Contract.

**Step 4**: Write failing acceptance/integration test.

**Step 5**: Write failing application test.

**Step 6**: Write failing domain test.

**Step 7**: Implement minimum code.

**Step 8**: Make tests pass.

**Step 9**: Refactor.

Never implement before tests exist.

---

## Rule 4

Protect Domain

Business Rules belong only inside Domain.

Never place business rules inside:

Controller

Infrastructure

Repository

Database

External Service

Application layer only coordinates.

---

## Rule 5

Every Aggregate must define:

- Purpose
- Aggregate Root
- Entities
- Value Objects
- Business Invariants
- Lifecycle
- Domain Events
- Repositories
- Factories
- Specifications

---

## Rule 6

Every Use Case must define:

- Business Goal
- Actor
- Trigger
- Preconditions
- Main Flow
- Alternative Flow
- Failure Flow
- Postconditions
- Acceptance Criteria

---

## Rule 7

Every implementation must begin by producing:

## Analysis

- Business Understanding
- Current Behavior
- Expected Behavior
- Affected Aggregates
- Affected APIs
- Affected Events
- Affected Database
- Risk
- Impact Analysis
- Implementation Plan

Do not implement before analysis is complete.

---

## Rule 8

Testing Requirements

Always create:

- Acceptance Test
- Integration Test
- Domain Test
- Application Test
- Regression Checklist

Test:

- Happy Path
- Validation
- Concurrency
- Rollback
- Idempotency
- Exception
- Edge Cases

---

## Rule 9

Documentation is part of the implementation.

Whenever code changes:

Update:

- Business Rules
- Workflow
- API
- Events
- Sequence Diagram
- Examples
- Database
- Architecture if required

Never leave documentation outdated.

---

## Rule 10

Definition of Done (Additional)

A task is NOT complete until:

✓ Business rules preserved
✓ Tests passing
✓ Documentation updated
✓ API documented
✓ Events documented
✓ Database documented
✓ Regression checklist updated
✓ No duplicated logic
✓ DDD respected
✓ SOLID respected
✓ Self review completed
✓ Multi-service impact analyzed (if applicable)
✓ Cross-service documentation updated (if applicable)
✓ Wion Engineering Rules followed
✓ Architecture remains clean
✓ No dead code left behind

---

# Self Review Checklist

Before finishing ask yourself:

- Can this business rule move into Domain?
- Can this Aggregate become inconsistent?
- Can this break another bounded context?
- Did I violate Aggregate boundaries?
- Did I introduce duplicated knowledge?
- Did I write enough tests?
- Can another developer understand this in six months?
- Is documentation synchronized?
- Did I follow all Wion Engineering Rules?
- Is the architecture cleaner than before?

If any answer is NO:

Continue improving before finishing.

---

# AI Navigation

## Knowledge Base Structure

```
.claude/
├── CLAUDE.md              # This file (Wion Engineering Rules)
├── playbooks/             # Global AI workflows
│   ├── new-feature.md
│   ├── bug-fix.md
│   ├── code-review.md
│   ├── documentation-update.md
│   └── refactoring.md
├── templates/             # Global templates
│   ├── business-rule.md
│   ├── policy.md
│   ├── api.md
│   ├── domain-event.md
│   ├── use-case.md
│   ├── adr.md
│   └── new-domain/
├── prompts/               # AI prompt templates
└── knowledge/             # Multi-service documentation
    ├── shared/            # Shared conventions
    │   ├── README.md
    │   ├── ddd-conventions.md
    │   ├── event-conventions.md
    │   ├── api-conventions.md
    │   ├── documentation-standards.md
    │   └── architecture-principles.md
    ├── wion-billing/      # WION Billing Service docs
    │   ├── README.md
    │   ├── architecture/
    │   ├── domains/
    │   ├── application/
    │   ├── infrastructure/
    │   ├── api/
    │   ├── policies/
    │   ├── events/
    │   ├── reference/
    │   └── adr/
    ├── billing-management/ # TMT Billing Management docs
    └── [other services]/  # Future services
```

---

## Task-Based Navigation

### For Single-Service Tasks

**Task**: "Implement wallet freeze feature"

1. **Discover service**: "Wallet" → `wion-billing`
2. **Load shared conventions**: `knowledge/shared/*.md`
3. **Load service docs**: `knowledge/wion-billing/`
4. **Load playbook**: `playbooks/new-feature.md`
5. **Execute**: Apply to service context

---

### For Multi-Service Tasks

**Task**: "Add payment webhook notifications"

1. **Discover services**: "Payment" → `wion-billing`, notification service
2. **Load shared conventions**: `knowledge/shared/*.md`
3. **Load billing docs**: `knowledge/wion-billing/`
4. **Cross-service analysis**: Identify integration points
5. **Execute**: Update all affected services

---

### For BuildingBlock Tasks

**Task**: "Add new validation helper"

1. **Check existing**: Search `src/framework/Wion.BuildingBlock`
2. **Reuse if found**: Improve existing implementation
3. **Create if new**: Add to appropriate BuildingBlock project
4. **Never use**: `src/building-blocks` (deprecated)

---

# Quick Reference

## Service Detection Patterns

| Indicator | Service Detection | Example |
|-----------|------------------|---------|
| **Explicit name** | Direct mention | "wion-billing" |
| **Domain concept** | Domain mapping | "Wallet" → wion-billing |
| **API endpoint** | Service mapping | `/api/v1/wallets/` → wion-billing |
| **Event name** | Service mapping | `PaymentSucceeded` → wion-billing |
| **File path** | Path parsing | `knowledge/wion-billing/` → wion-billing |

---

## Architecture Patterns

### Repository Pattern (TMT Style)
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

### Aggregate Pattern (DDD Style)
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

# Support

## Architecture Team

**Questions about Wion engineering architecture**?
- **Architecture Team**: architecture@wion.vn
- **Tech Lead**: tech-lead@wion.vn

---

# Version

**Knowledge Base Version**: 3.0 (Wion Engineering Rules + Multi-Service Architecture)  
**Last Updated**: 2026-07-24  
**Migration**: From v2.0 to v3.0 - Added Wion Engineering Rules