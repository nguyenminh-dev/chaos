# Billing Service - Glossary

## Domain Model Terms

### Bounded Context
**Definition**: The specific domain boundary within which a particular domain model applies

**Billing Service Context**: Financial Operations bounded context

**Scope**: All monetary transactions, credit management, wallet operations, and financial reporting

**Ubiquitous Language**: Wallet, Wi Credit, Balance, Ledger, Payment, Invoice

---

### Aggregate
**Definition**: A cluster of domain objects that can be treated as a single unit

**Characteristics**:
- Has one Aggregate Root (entry point)
- Enforces business invariants within boundary
- Maintains transaction consistency
- External access only through Aggregate Root

**Examples**:
- Wallet Aggregate (Wallet + WalletAssets)
- Payment Aggregate (Payment + Webhooks)
- CreditTransaction Aggregate

**Related**: [Service Documentation](./service.md)

---

### Aggregate Root
**Definition**: The single entity that provides access to all objects within the Aggregate

**Rules**:
- Only Aggregate Root can be accessed externally
- All operations go through Aggregate Root
- Enforces consistency boundary
- Has unique identity

**Examples**:
- `Wallet` - Root of Wallet Aggregate
- `Payment` - Root of Payment Aggregate
- `CreditTransaction` - Root of Credit Transaction Aggregate

---

### Value Object
**Definition**: An immutable object that represents a domain concept

**Characteristics**:
- No identity (identified by attributes)
- Immutable (cannot be modified after creation)
- Can be shared across Aggregates

**Examples**:
- `Balance` - Immutable balance representation
- `Currency` - Currency type (VND)
- `Money` - Amount + Currency combination
- `AssetType` - Asset classification

---

### Domain Event
**Definition**: Something that happened in the domain that domain experts care about

**Purpose**:
- Notify other parts of the system
- Maintain eventual consistency
- Enable async processing

**Examples**:
- `WalletCreated` - New wallet created
- `BalanceChanged` - Balance updated
- `PaymentSucceeded` - Payment completed

**Related**: [Event Index](./event-index.md)

---

### Domain Service
**Definition**: Stateless service that performs operations that don't naturally belong to a single Aggregate

**Use Cases**:
- Operations spanning multiple Aggregates
- Calculations requiring external services
- Complex business logic

**Examples**:
- `BalanceCalculationService` - Calculates total balance across assets
- `LedgerValidationService` - Validates double-entry accounting

---

### Repository
**Definition**: Interface that mediates between Domain and data mapping

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

---

### Specification
**Definition**: Business rule encapsulated as a predicate

**Purpose**:
- Encapsulate business query logic
- Make business rules explicit
- Enable reusable business conditions

**Examples**:
- `ActiveWalletSpec` - "Wallet is not deleted"
- `SufficientBalanceSpec` - "Balance >= required amount"
- `ExpiredPaymentSpec` - "Payment expired (timeout)"

---

### Invariant
**Definition**: A rule that must always be true for an Aggregate

**Enforcement**:
- Checked in Aggregate methods
- Cannot be violated by any operation
- Maintained within transaction boundary

**Examples**:
- `balance >= 0` - Wallet balance invariant
- `Σ debit = Σ credit` - Ledger double-entry invariant
- `status transitions are valid` - Payment state machine invariant

---

### Transaction Boundary
**Definition**: The scope of a single database transaction

**DDD Rule**: One Aggregate = One Transaction

**Implications**:
- Never modify multiple Aggregates in one transaction
- Use Domain Events for cross-Aggregate coordination
- Accept eventual consistency for cross-Aggregate operations

**Example**:
- Credit consumption updates Wallet, CreditTransaction, and Ledger
- Invoice creation happens in separate transaction
- Triggered by `PaymentSucceeded` event

---

### Ubiquitous Language
**Definition**: Shared language used by both developers and domain experts

**Purpose**:
- Bridge communication gap
- Ensure terminology consistency
- Prevent misunderstandings

**Billing Service Terms**:
- Wallet - Digital wallet for tenant
- Wi Credit - Internal payment unit
- Balance - Available funds
- Reserved Balance - Locked funds
- Ledger - Accounting record
- Invoice - Electronic invoice document

---

## Business Terms

### Wi Credit
**Definition**: Internal payment unit of WION Platform

**Conversion Rate**: 1 VNĐ = 1 Wi Credit (configurable)

**Constraints**:
- Cannot reverse convert to cash
- Can be consumed across all WION products

**Related**: [Wallet Management](./features/wallet-management/overview.md)

---

### Wallet
**Definition**: Digital wallet belonging to a tenant that stores balance and assets

**Context**: Each tenant has exactly one wallet

**Components**: Available balance + Reserved balance

**Related**: [Wallet Management](./features/wallet-management/overview.md)

---

### Reserved Balance
**Definition**: Funds temporarily locked for pending transactions

**Purpose**: Prevent double-spending during transaction processing

**Behavior**: Released back to available balance on transaction completion

**Related**: [Credit Consumption](./features/credit-consumption/overview.md)

---

### Asset Type
**Definition**: Different types of credits stored in wallet

**Values**:
- `WI_CREDIT`: Standard WION credits
- `PROMOTION`: Promotional credits with expiration
- `GIFT`: Gift credits from campaigns
- `TRIAL`: Trial credits for new users
- `AI_TOKEN`: Future - AI service tokens

**Related**: [Wallet Management](./features/wallet-management/overview.md)

---

### Double-Entry Accounting
**Definition**: Accounting method where every transaction has equal debit and credit entries

**Rule**: Σ Debit = Σ Credit

**Purpose**: Ensure transaction integrity and auditability

**Related**: [Credit Consumption](./features/credit-consumption/overview.md)

---

### Idempotency Key
**Definition**: Unique identifier for a transaction request

**Purpose**: Prevent duplicate transaction processing

**Scope**: Unique per transaction attempt

**Usage**: All mutation operations require idempotency key

**Related**: [Credit Consumption](./features/credit-consumption/overview.md)

---

### Reference ID
**Definition**: External system's transaction identifier

**Purpose**: Link Billing transactions to external systems

**Examples**: Payment gateway ID, Invoice number, Service request ID

**Constraint**: Unique per source system

---

## Clean Architecture Terms

### Clean Architecture
**Definition**: Layered architecture where dependencies point INWARD

**Principle**: Business logic (inner layers) independent of technical details (outer layers)

**Layers**:
- **Domain Layer** (innermost) - Business rules, zero dependencies
- **Application Layer** - Use cases, orchestration
- **API Layer** - Controllers, endpoints
- **Infrastructure Layer** (outermost) - Databases, external services

**Dependency Rule**: Outer layers depend on inner layers only

---

### Business Rule Protection
**Principle**: Business rules belong ONLY in Domain Layer

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
**Definition**: Command Query Responsibility Segregation

**Pattern**: Separate models for read (Query) and write (Command) operations

**Benefits**:
- Optimize read model for queries (caching, denormalization)
- Optimize write model for consistency (transactional)

**Application in Billing Service**:
- `ConsumeCreditCommand` - Transactional write operation
- `GetBalanceQuery` - Optimized read operation (cache-first)

---

### Event-Driven Architecture
**Definition**: Architecture where components communicate via events

**Characteristics**:
- Asynchronous communication
- Loose coupling
- Eventual consistency
- At-least-once delivery

**Types**:
- **Domain Events** - Published when Aggregate state changes
- **Integration Events** - Cross-service communication

---

## Technical Terms

### API Key
**Definition**: Authentication token for API access

**Usage**: Sent in `Authorization: Bearer {api-key}` header

**Scope**: Per-application or per-tenant

---

### HMAC-SHA256
**Definition**: Cryptographic signature algorithm for webhook verification

**Purpose**: Verify webhook authenticity and integrity

**Implementation**: `signature = HMAC-SHA256(secret_key, payload_json)`

**Related**: [Webhook Handling](./features/webhook-handling/overview.md)

---

### P95 Latency
**Definition**: 95th percentile response time

**Meaning**: 95% of requests complete within this time

**Usage**: Performance SLA metric

**Targets**:
- Balance check: < 100ms
- Credit consume: < 100ms
- Payment initiation: < 500ms

---

### TTL (Time To Live)
**Definition**: Cache expiration duration

**Usage**: Redis cache stores data for specified duration

**Default**: 60 seconds for balance data

---

### Dead Letter Queue
**Definition**: Queue for failed events/messages

**Purpose**: Store events that failed processing for retry or manual intervention

**Retry**: 3 attempts with exponential backoff before DLQ

---

### Soft Delete
**Definition**: Marking records as deleted without physical deletion

**Implementation**: Set `is_deleted = TRUE` flag

**Purpose**: Maintain audit trail and data integrity

**Retention**: 7 years for compliance

---

## Acronyms

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| API | Application Programming Interface | Service endpoints |
| NFR | Non-Functional Requirement | Performance, security requirements |
| TPS | Transactions Per Second | Throughput metric |
| SLA | Service Level Agreement | Service quality commitment |
| RTO | Recovery Time Objective | Max downtime after disaster |
| RPO | Recovery Point Objective | Max data loss after disaster |
| TTL | Time To Live | Cache expiration duration |
| HMAC | Keyed-Hash Message Authentication Code | Signature algorithm |
| DLQ | Dead Letter Queue | Failed message queue |
| P95 | 95th Percentile | Performance metric |
| DDD | Domain-Driven Design | Software development approach |
| CQRS | Command Query Responsibility Segregation | Read/write separation pattern |
| EDA | Event-Driven Architecture | Event-based communication |

## External System Terms

### TPayGate
**Definition**: Payment gateway provider for QR code payments

**Role**: Process QR payments and send webhooks

**Integration**: REST API + Webhooks

**Related**: [Payment Processing](./features/payment-processing/overview.md)

---

### Invoice Hub
**Definition**: Electronic invoice generation system

**Role**: Issue electronic invoices per Vietnam regulations

**Integration**: REST API

**Related**: [Invoice Generation](./features/invoice-generation/overview.md)

---

## Related Documentation
- [Service Documentation](./service.md)
- [Feature Index](./feature-index.md)
