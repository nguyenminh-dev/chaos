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

**Do NOT duplicate** shared concepts in service documentation. Instead, reference these shared documents.

---

## Shared Documentation

### Engineering Standards

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
| **DDD patterns** | `shared/ddd-conventions.md` | All services |
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
1. Load shared conventions first
2. Load service-specific documentation
3. Apply shared patterns to service context

**Example workflow**:
```bash
# Step 1: Load shared conventions
load(shared/ddd-conventions.md)
load(shared/api-conventions.md)

# Step 2: Load service docs
load(knowledge/{service}/README.md)
load(knowledge/{service}/domains/)

# Step 3: Apply conventions
"This service follows event naming: {Aggregate}{StateChange}"
```

---

## Maintaining Shared Documentation

### When to Update Shared Docs

**Update shared documentation when**:
- ✅ New universal pattern discovered
- ✅ Existing convention needs clarification
- ✅ Cross-service requirement emerges
- ✅ Architectural decision affects all services

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

---

## Conventions by Category

### Domain-Driven Design

**Location**: `ddd-conventions.md`

**Key conventions**:
- Aggregate design patterns
- Business rule format
- Domain event naming
- Repository patterns
- Policy pattern

**Used by**: All services with domain logic

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

**Current version**: 1.0

**Versioning rules**:
- Major version: Breaking changes
- Minor version: Additions, compatible changes
- Patch version: Fixes, clarifications

**Example**:
```
1.0 → 1.1  (Minor: Added new pattern)
1.1 → 2.0  (Major: Breaking change)
1.0 → 1.0.1  (Patch: Fixed typo)
```

---

## Related Documentation

### Service-Specific Documentation

Each service has its own Knowledge Base:
```
knowledge/
├── shared/              # This directory
├── billing-service/     # Billing Service docs
├── tenant-service/      # Tenant Service docs
└── [other services]/    # Future services
```

**Service-specific docs** should:
- ✅ Reference shared conventions
- ✅ Document service-specific patterns
- ❌ NOT duplicate shared knowledge

---

## Support

**Questions about shared conventions**?
- **Architecture Team**: architecture@wion.vn
- **Tech Lead**: tech-lead@wion.vn

---

## Index

### Quick Links

- [DDD Conventions](./ddd-conventions.md)
- [Event Conventions](./event-conventions.md)
- [API Conventions](./api-conventions.md)

---

**Last Updated**: 2026-07-05  
**Maintained By**: Architecture Team  
**Version**: 1.0
