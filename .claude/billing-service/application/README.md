# Application Layer - Use Cases

This directory contains the application layer of the Billing Service, organizing use cases and application workflows.

## Purpose
The application layer orchestrates domain operations to fulfill user goals. It contains NO business logic—only coordination.

## Key Principles

### Use Cases
- Represent user workflows
- Coordinate multiple aggregates
- Handle transaction boundaries
- Delegate to domain for business logic

### Application Services
- Orchestrate domain operations
- Handle infrastructure interactions
- Publish domain events
- Manage external service calls

### Event Handlers
- React to domain events
- Trigger policies
- Maintain eventual consistency

## Use Cases

### [Consume Credit](./use-cases/consume-credit.md)
**Business Goal**: Enable products to consume credits with real-time balance updates

**Actor**: Application (WION POS, FnB, SPA, etc.)

**Aggregates Involved**:
- Wallet (balance management)
- CreditTransaction (consumption tracking)
- Ledger (audit trail)

**Key Workflows**:
- Idempotency checking
- Balance reservation
- Service request processing
- Balance confirmation
- Ledger entry creation

---

### [Process Payment](./use-cases/process-payment.md)
**Business Goal**: Enable tenants to topup credits through QR code payments

**Actor**: Tenant/User

**Aggregates Involved**:
- Payment (payment tracking)
- Wallet (balance credit on success)
- Ledger (accounting)

**Key Workflows**:
- Payment initiation
- TPayGate integration
- QR code generation
- Webhook handling

---

### [Handle Webhook](./use-cases/handle-webhook.md)
**Business Goal**: Process payment gateway callbacks securely

**Actor**: TPayGate

**Aggregates Involved**:
- Payment (status update)
- Wallet (balance credit)
- Ledger (accounting)

**Key Workflows**:
- Signature verification
- Duplicate detection
- Payment status update
- Wallet crediting
- Invoice triggering

---

### [Create Invoice](./use-cases/create-invoice.md)
**Business Goal**: Automatically issue invoices on successful payments

**Actor**: System (triggered by event)

**Aggregates Involved**:
- Payment (payment details)
- InvoiceReference (invoice tracking)
- Ledger (accounting)

**Key Workflows**:
- Invoice Hub API integration
- Invoice number generation
- Retry handling
- Manual processing queue

## Application Layer Characteristics

### No Business Logic
- Business rules live in Domain
- Application only coordinates
- Delegates to aggregates for logic

### Transaction Management
- Maintains aggregate boundaries
- One aggregate per transaction
- Eventual consistency between aggregates

### Infrastructure Orchestration
- Database transactions
- External service calls
- Event publishing
- Caching strategy

## Error Handling
- Domain errors: Return to user with clear message
- Infrastructure errors: Retry or fail gracefully
- Event processing: Dead letter queue
- Idempotency: Safe retries

## Performance Requirements
- **Consume Credit**: 10,000 TPS
- **Process Payment**: < 500ms QR code generation
- **Handle Webhook**: < 1 second processing
- **Create Invoice**: Async processing with retries

## Testing Strategy
- Acceptance tests for user workflows
- Integration tests for external services
- Contract tests for API boundaries
- Event-driven tests for async flows
