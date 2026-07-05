# DDD & Architecture Glossary

**Purpose**: Universal terminology for Domain-Driven Design and Clean Architecture concepts used across all WION microservices.

**Scope**: These definitions apply to **ALL services**. Never duplicate these definitions in service-specific documentation.

---

## Domain-Driven Design Terms

### Bounded Context
**Definition**: The specific domain boundary within which a particular domain model applies.

**Characteristics**:
- Has its own ubiquitous language
- Maintains consistency within boundaries
- Integrates with other contexts via shared kernels or anti-corruption layers

**Example**: Billing Service operates within **Financial Operations** bounded context.

---

### Aggregate
**Definition**: A cluster of domain objects that can be treated as a single unit.

**Characteristics**:
- Has one Aggregate Root (entry point)
- Enforces business invariants within boundary
- Maintains transaction consistency
- External access only through Aggregate Root

**Examples**:
- Wallet Aggregate (Wallet + WalletAssets)
- Payment Aggregate (Payment + PaymentWebhooks)
- Order Aggregate (Order + OrderItems)

**Related**: [DDD Conventions](./ddd-conventions.md)

---

### Aggregate Root
**Definition**: The single entity that provides access to all objects within the Aggregate.

**Rules**:
- Only Aggregate Root can be accessed externally
- All operations go through Aggregate Root
- Enforces consistency boundary
- Has unique identity

**Examples**:
- `Wallet` - Root of Wallet Aggregate
- `Payment` - Root of Payment Aggregate
- `Order` - Root of Order Aggregate

---

### Value Object
**Definition**: An immutable object that represents a domain concept.

**Characteristics**:
- No identity (identified by attributes)
- Immutable (cannot be modified after creation)
- Can be shared across Aggregates
- Replaceable rather than mutable

**Examples**:
- `Balance` - Immutable balance representation
- `Currency` - Currency type (VND, USD)
- `Money` - Amount + Currency combination
- `EmailAddress` - Validated email address

---

### Domain Event
**Definition**: Something that happened in the domain that domain experts care about.

**Purpose**:
- Notify other parts of the system
- Maintain eventual consistency
- Enable async processing
- Preserve audit trail

**Examples**:
- `WalletCreated` - New wallet created
- `BalanceChanged` - Balance updated
- `PaymentSucceeded` - Payment completed
- `OrderPlaced` - Order created

**Related**: [Event Conventions](./event-conventions.md)

---

### Domain Service
**Definition**: Stateless service that performs operations that don't naturally belong to a single Aggregate.

**Use Cases**:
- Operations spanning multiple Aggregates
- Calculations requiring external services
- Complex business logic that doesn't fit in one Aggregate

**Examples**:
- `BalanceCalculationService` - Calculates total balance across assets
- `LedgerValidationService` - Validates double-entry accounting
- `TaxCalculationService` - Computes tax for orders

---

### Repository
**Definition**: Interface that mediates between Domain and data mapping.

**Purpose**:
- Abstract persistence mechanism
- Provide collection-like access to Aggregates
- Hide database details from Domain

**Rules**:
- Defined in Domain Layer as interface
- Implemented in Infrastructure Layer
- Returns Aggregates, not data structures

**Examples**:
- `IWalletRepository` - Load/save Wallet Aggregates
- `IPaymentRepository` - Load/save Payment Aggregates
- `IOrderRepository` - Load/save Order Aggregates

---

### Specification
**Definition**: Business rule encapsulated as a predicate.

**Purpose**:
- Encapsulate business query logic
- Make business rules explicit
- Enable reusable business conditions

**Examples**:
- `ActiveWalletSpec` - "Wallet is not deleted"
- `SufficientBalanceSpec` - "Balance >= required amount"
- `ExpiredPaymentSpec` - "Payment expired (timeout)"
- `ValidCustomerSpec` - "Customer is in good standing"

---

### Invariant
**Definition**: A rule that must always be true for an Aggregate.

**Enforcement**:
- Checked in Aggregate methods
- Cannot be violated by any operation
- Maintained within transaction boundary

**Examples**:
- `balance >= 0` - Wallet balance invariant
- `Σ debit = Σ credit` - Ledger double-entry invariant
- `status transitions are valid` - Payment state machine invariant
- `order total >= 0` - Order total invariant

---

### Transaction Boundary
**Definition**: The scope of a single database transaction.

**DDD Rule**: One Aggregate = One Transaction

**Implications**:
- Never modify multiple Aggregates in one transaction
- Use Domain Events for cross-Aggregate coordination
- Accept eventual consistency for cross-Aggregate operations

**Example**:
- ❌ **WRONG**: Update Wallet + Payment + Ledger in one transaction
- ✅ **RIGHT**: Update Wallet, publish event, handler updates Ledger

---

### Ubiquitous Language
**Definition**: Shared language used by both developers and domain experts.

**Purpose**:
- Bridge communication gap
- Ensure terminology consistency
- Prevent misunderstandings

**Example Terms**:
- Wallet - Digital wallet for tenant
- Order - Customer purchase order
- Invoice - Electronic invoice document
- Shipment - Physical delivery

---

## Clean Architecture Terms

### Clean Architecture
**Definition**: Layered architecture where dependencies point INWARD.

**Principle**: Business logic (inner layers) independent of technical details (outer layers).

**Layers**:
- **Domain Layer** (innermost) - Business rules, zero dependencies
- **Application Layer** - Use cases, orchestration
- **API Layer** - Controllers, endpoints
- **Infrastructure Layer** (outermost) - Databases, external services

**Dependency Rule**: Outer layers depend on inner layers only

---

### Business Rule Protection
**Principle**: Business rules belong ONLY in Domain Layer.

**Violations** (Anti-patterns):
- ❌ Business rules in Controllers
- ❌ Business rules in Services (without domain logic)
- ❌ Business rules in stored procedures
- ❌ Business rules in database constraints/triggers

**Correct Approach**:
- ✓ Business rules in Aggregates
- ✓ Business rules in Value Objects
- ✓ Business rules in Domain Services

---

### CQRS
**Definition**: Command Query Responsibility Segregation.

**Pattern**: Separate models for read (Query) and write (Command) operations.

**Benefits**:
- Optimize read model for queries (caching, denormalization)
- Optimize write model for consistency (transactional)

**Example**:
- `ConsumeCreditCommand` - Transactional write operation
- `GetBalanceQuery` - Optimized read operation (cache-first)

---

### Event-Driven Architecture
**Definition**: Architecture where components communicate via events.

**Characteristics**:
- Asynchronous communication
- Loose coupling
- Eventual consistency
- At-least-once delivery

**Types**:
- **Domain Events** - Published when Aggregate state changes
- **Integration Events** - Cross-service communication

**Related**: [Event Conventions](./event-conventions.md)

---

## Common Acronyms

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **DDD** | Domain-Driven Design | Software development approach |
| **CQRS** | Command Query Responsibility Segregation | Read/write separation pattern |
| **EDA** | Event-Driven Architecture | Event-based communication |
| **API** | Application Programming Interface | Service endpoints |
| **NFR** | Non-Functional Requirement | Performance, security requirements |
| **TPS** | Transactions Per Second | Throughput metric |
| **SLA** | Service Level Agreement | Service quality commitment |
| **RTO** | Recovery Time Objective | Max downtime after disaster |
| **RPO** | Recovery Point Objective | Max data loss after disaster |
| **HMAC** | Keyed-Hash Message Authentication Code | Signature algorithm |
| **DLQ** | Dead Letter Queue | Failed message queue |
| **P95** | 95th Percentile | Performance metric |
| **TTL** | Time To Live | Cache expiration duration |

---

## Usage Guidelines

### For Service Documentation

**DO**: Reference shared glossary
```markdown
## Domain Model

### Aggregate
Follows standard [DDD definition](../../../shared/glossary.md#aggregate).

### Wallet Aggregate
{Service-specific details}
```

**DON'T**: Duplicate shared definitions
```markdown
## Domain Model

### Aggregate
Definition: A cluster of domain objects...  ❌ WRONG (duplicates shared docs)
```

---

### For AI Agents

**When working on a service**:
1. Load shared glossary first
2. Use universal definitions consistently
3. Add service-specific terms in service glossary

**Example workflow**:
```bash
# Step 1: Load shared glossary
load(knowledge/shared/glossary.md)

# Step 2: Load service-specific glossary
load(knowledge/{service}/reference/glossary.md)

# Step 3: Apply terminology consistently
"Wallet Aggregate follows standard Aggregate pattern"
```

---

## Service-Specific Glossaries

Each service should have its own glossary for service-specific terms:

```
knowledge/
├── shared/
│   └── glossary.md          # This file (universal DDD terms)
└── {service}/
    └── reference/
        └── glossary.md      # Service-specific terms
```

**Service glossaries should**:
- ✅ Define service-specific business terms
- ✅ Reference shared definitions for universal concepts
- ❌ NOT duplicate universal DDD definitions

---

## Related Documentation

- [DDD Conventions](./ddd-conventions.md) - DDD patterns and principles
- [Event Conventions](./event-conventions.md) - Event patterns
- [API Conventions](./api-conventions.md) - API patterns

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-07-05 | Initial version with universal DDD and architecture terms |

---

**Last Updated**: 2026-07-05
**Maintained By**: Architecture Team
**Version**: 1.0
