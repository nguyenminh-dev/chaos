# Domain-Driven Design Conventions

**Purpose**: Standard DDD conventions and patterns used across all WION microservices.

**Scope**: These conventions apply to **all services** in the WION ecosystem. Do not duplicate these in service-specific documentation.

---

## Core DDD Principles

### 1. Ubiquitous Language

**Definition**: A shared, rigorous language used by both developers and domain experts.

**Convention**:
- ✅ Use domain terminology consistently across code and documentation
- ✅ Create a glossary for each bounded context
- ✅ Avoid technical jargon in domain discussions
- ✅ Define terms in `reference/glossary.md` for each service

**Example**:
```markdown
✅ CORRECT:
"Wallet balance cannot go negative"
"Payment transitions from PENDING to COMPLETED"

❌ WRONG:
"The database field can't be below zero"
"The status column changes from 1 to 2"
```

---

### 2. Bounded Contexts

**Definition**: A distinct part of the domain logic with well-defined boundaries.

**Convention**:
- ✅ Each microservice represents one bounded context
- ✅ Define context ownership clearly
- ✅ Document upstream/downstream relationships
- ✅ Use context mapping for integration

**Structure**:
```
{Service}/architecture/bounded-context.md
```

**Example**:
```markdown
## Bounded Context
**Service**: Billing Service
**Context**: Financial Operations
**Responsibilities**: Payments, wallets, credits, invoices
```

---

### 3. Aggregates

**Definition**: A cluster of domain objects that can be treated as a unit.

**Convention**:
- ✅ Each aggregate has exactly one **Aggregate Root**
- ✅ Aggregates maintain **consistency boundaries**
- ✅ No cross-aggregate transactions
- ✅ Eventual consistency between aggregates

**Structure**:
```
{Service}/domains/{domain}/aggregate.md
```

**Example**:
```markdown
## Wallet Aggregate
**Aggregate Root**: Wallet
**Entities**: Wallet (Root), WalletAsset
**Transaction Boundary**: Single wallet per transaction
```

---

### 4. Value Objects

**Definition**: An immutable object that represents a descriptive aspect of the domain with no identity.

**Convention**:
- ✅ Value objects are immutable
- ✅ Value objects have no identity
- ✅ Value objects can be shared
- ✅ Document in `domains/{domain}/model.md`

**Examples**:
```typescript
✅ CORRECT:
class Balance {
  readonly amount: number;
  readonly currency: Currency;
  // No ID - value object
}

class Money {
  readonly value: number;
  readonly currency: string;
  // Immutable, no ID
}
```

---

### 5. Domain Events

**Definition**: Events that represent something that happened in the domain.

**Convention**:
- ✅ Use past tense naming: `WalletCreated`, `PaymentSucceeded`
- ✅ Events are immutable
- ✅ Events contain minimal data (IDs + relevant state)
- ✅ Document in `domains/{domain}/domain-events.md`

**Structure**:
```
{Service}/domains/{domain}/domain-events.md
{Service}/events/README.md (catalog)
```

**Examples**:
```typescript
✅ CORRECT:
interface WalletCreated {
  eventType: "WalletCreated";
  walletId: string;
  tenantId: string;
  createdAt: Date;
}

❌ WRONG:
interface CreateWallet {
  // Should be past tense
  wallet: Wallet; // Too much data
}
```

---

### 6. Repositories

**Definition**: Collections-like interfaces for accessing aggregates.

**Convention**:
- ✅ Repositories access **Aggregates**, not entities
- ✅ Repository interfaces defined in domain
- ✅ Repository implementations in infrastructure
- ✅ Document in `domains/{domain}/repositories.md`

**Structure**:
```
Domain: {Service}/domains/{domain}/repositories.md
Implementation: {Service}/infrastructure/database.md
```

**Example**:
```typescript
// Domain interface (in domain/)
interface IWalletRepository {
  save(wallet: Wallet): Promise<void>;
  findById(tenantId: string): Promise<Wallet | null>;
}

// Implementation (in infrastructure/)
class PostgresWalletRepository implements IWalletRepository {
  // Implementation
}
```

---

### 7. Specifications

**Definition**: Encapsulated business rules for querying.

**Convention**:
- ✅ Specifications encapsulate business logic
- ✅ Specifications are composable
- ✅ Document in aggregate files

**Example**:
```typescript
class ActiveWalletSpec {
  isSatisfiedBy(wallet: Wallet): boolean {
    return wallet.deletedAt === null;
  }
}

class SufficientBalanceSpec {
  constructor(private requiredAmount: number) {}
  isSatisfiedBy(wallet: Wallet): boolean {
    return wallet.availableBalance >= this.requiredAmount;
  }
}
```

---

## Aggregate Design Rules

### Rule 1: Aggregate Root Identity

**Every aggregate must have a unique identifier**.

**Convention**:
```typescript
✅ CORRECT:
class Wallet {
  readonly id: string;  // Unique ID
  // ...
}

❌ WRONG:
class Wallet {
  // No ID - not an aggregate root
}
```

---

### Rule 2: Aggregate Boundaries

**Aggregates maintain consistency within their boundaries**.

**Convention**:
- ✅ All invariants enforced within aggregate
- ✅ No cross-aggregate references (use IDs only)
- ✅ Eventual consistency between aggregates

**Example**:
```typescript
✅ CORRECT:
class Payment {
  // References wallet by ID
  walletId: string;  // Not Wallet object
}

❌ WRONG:
class Payment {
  // Direct reference - breaks aggregate boundary
  wallet: Wallet;
}
```

---

### Rule 3: Aggregate Size

**Aggregates should be small**.

**Convention**:
- ✅ One aggregate root + few entities
- ✅ Keep aggregate footprint small
- ✅ Design for transactional consistency

**Example**:
```typescript
✅ CORRECT:
Wallet (root)
├── WalletAsset (entity)
└── Balance (value object)

❌ WRONG:
Wallet (root)
├── WalletAsset (entity)
├── PaymentHistory (entity)
├── Transaction (entity)
├── InvoiceReference (entity)
└── ... (too large)
```

---

## Domain Event Conventions

### Naming Convention

**Format**: `{Aggregate}{StateChange}`

**Examples**:
- ✅ `WalletCreated` - Wallet created
- ✅ `BalanceChanged` - Balance updated
- ✅ `PaymentSucceeded` - Payment completed
- ✅ `InsufficientBalance` - Balance validation failed

---

### Event Payload Convention

**Structure**:
```typescript
interface DomainEvent {
  eventType: string;        // Event name
  aggregateId: string;       // Aggregate root ID
  occurredAt: Date;         // When it happened
  data: EventData;          // Event-specific data
}
```

**Example**:
```typescript
✅ CORRECT:
interface BalanceChanged {
  eventType: "BalanceChanged";
  aggregateId: string;      // walletId
  occurredAt: Date;
  data: {
    oldBalance: number;
    newBalance: number;
    changeAmount: number;
  };
}

❌ WRONG:
interface BalanceChanged {
  eventType: "BalanceChanged";
  data: {
    wallet: Wallet;  // Too much data
    timestamp: string;  // Should be Date
  };
}
```

---

## Business Rule Conventions

### Rule ID Format

**Format**: `BR-{DOMAIN}-{NUMBER}`

**Examples**:
- `BR-W-001` - Wallet rule #1
- `BR-P-003` - Payment rule #3
- `BR-C-007` - Credit Transaction rule #7

**Domain Letters**:
- `W` - Wallet
- `P` - Payment
- `C` - Credit Transaction
- `L` - Ledger
- `I` - Invoice
- `O` - Order
- `U` - User
- `T` - Tenant

---

### Rule Definition Template

**Every business rule must have**:

```markdown
### BR-{DOMAIN}-{NUMBER}: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical/logical definition}
**Enforcement Points**: {Where enforced}
**Violation Handling**: {What happens on violation}
**Purpose**: {Why this rule exists}
```

**Example**:
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.
**Formal Definition**: availableBalance >= 0
**Enforcement Points**: 
- debit(amount) operation
- Database constraint
**Violation Handling**: 
- Throw InsufficientBalanceException
- Publish InsufficientBalance event
**Purpose**: Maintain financial integrity
```

---

## Cross-Aggregate Coordination

### Policy Pattern

**When**: Business logic spans multiple aggregates.

**Convention**:
- ✅ Create **Policy** for cross-aggregate logic
- ✅ Policies are **event-driven**
- ✅ Policies maintain **aggregate independence**
- ✅ Document in `policies/{policy}.md`

**Structure**:
```
{Service}/policies/{policy}.md
```

**Example**:
```markdown
# Invoice After Payment Policy

## Trigger
PaymentSucceeded event from Payment Aggregate

## Participating Aggregates
- Payment (source)
- Invoice (target)

## Flow
1. Receive PaymentSucceeded event
2. Create InvoiceReference aggregate
3. Call Invoice Hub API
4. Update invoice status
```

---

## Documentation Conventions

### Domain Documentation Structure

**Every domain must have**:

```
{Service}/domains/{domain}/
├── overview.md          # Domain purpose and scope
├── aggregate.md         # Aggregate root and entities
├── model.md             # Value objects and types
├── business-rules.md    # Business invariants
├── lifecycle.md         # State transitions
├── domain-events.md     # Published events
└── repositories.md      # Repository interfaces (if applicable)
```

---

### Use Case Documentation Structure

**Every use case must have**:

```
{Service}/application/use-cases/{use-case}.md

## Business Goal
## Actor
## Trigger
## Preconditions
## Domain References        # CRITICAL: Reference domain, don't duplicate
### Aggregates Involved
### Business Rules Enforced
### Policies Applied
## Main Flow
## Alternative Flow
## Failure Flow
## Postconditions
## Acceptance Criteria
## Related APIs
## Related Events
```

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Anemic Domain Model

**Wrong**: Business logic in services or repositories

**Correct**: Business logic in domain aggregates

---

### ❌ Anti-Pattern 2: Cross-Aggregate Transactions

**Wrong**: Transactions spanning multiple aggregates

**Correct**: Eventual consistency via domain events

---

### ❌ Anti-Pattern 3: God Aggregates

**Wrong**: Aggregates with 10+ entities

**Correct**: Small, focused aggregates

---

### ❌ Anti-Pattern 4: Identifying Aggregates by Database Tables

**Wrong**: One table = one aggregate

**Correct**: Aggregates defined by business consistency boundaries

---

### ❌ Anti-Pattern 5: Domain Events for Everything

**Wrong**: Events for every state change

**Correct**: Events only for significant business events

---

## Quick Reference

### "Where do I document X?"

| Concept | Location | Shared? |
|---------|----------|---------|
| **DDD Principles** | `knowledge/shared/ddd-conventions.md` | ✅ Yes |
| **Aggregate** | `{Service}/domains/{domain}/aggregate.md` | ❌ No |
| **Business Rules** | `{Service}/domains/{domain}/business-rules.md` | ❌ No |
| **Domain Events** | `{Service}/domains/{domain}/domain-events.md` | ❌ No |
| **Use Cases** | `{Service}/application/use-cases/` | ❌ No |
| **Policies** | `{Service}/policies/` | ❌ No |

---

## Related Documentation

- [Event Conventions](./event-conventions.md) - Domain event patterns
- [API Conventions](./api-conventions.md) - API design patterns
- [Documentation Standards](./documentation-standards.md) - Documentation structure
- [Architecture Principles](./architecture-principles.md) - System architecture

---

**Convention Version**: 1.0  
**Last Updated**: 2026-07-05  
**Applies To**: All WION microservices
