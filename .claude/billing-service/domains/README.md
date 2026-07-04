# Domains - Business Knowledge

This directory contains the business knowledge of the Billing Service, organized by domain following Domain-Driven Design (DDD) principles.

## Purpose
Each domain represents a core business concept with:
- **Single Source of Truth**: Business rules, invariants, and lifecycle
- **Aggregate Boundary**: Transaction consistency
- **Ubiquitous Language**: Shared vocabulary

## Domains

### [Wallet Domain](./wallet/)
**Purpose**: Manage tenant wallets and balances

**Aggregate Root**: Wallet

**Key Concepts**:
- Multi-asset support (Wi Credit, Promotion, Gift, Trial)
- Balance management with reserved funds
- Per-tenant wallet isolation

**Business Rules**:
- Balance cannot be negative
- One wallet per tenant
- Reserved balance cannot be negative

---

### [Payment Domain](./payment/)
**Purpose**: Track payment transactions from initiation to completion

**Aggregate Root**: Payment

**Key Concepts**:
- QR code payment via TPayGate
- Payment lifecycle management
- Webhook handling

**Business Rules**:
- Payment amount must be positive
- Completed payments cannot be modified
- Expired payments auto-cancel

---

### [Credit Transaction Domain](./credit-transaction/)
**Purpose**: Handle credit consumption and refunds

**Aggregate Root**: CreditTransaction

**Key Concepts**:
- Idempotent credit operations
- Refund handling
- Balance adjustments

**Business Rules**:
- Transaction amount must be positive
- Idempotency key must be unique
- Refund cannot exceed original amount

---

### [Ledger Domain](./ledger/)
**Purpose**: Maintain double-entry accounting for audit trail

**Aggregate Root**: LedgerEntry

**Key Concepts**:
- Double-entry accounting
- Immutable ledger entries
- Transaction audit trail

**Business Rules**:
- Every transaction must have equal debit and credit: Σ Debit = Σ Credit
- Ledger entries are immutable (append-only)
- All financial operations must create ledger entries

---

### [Invoice Domain](./invoice/)
**Purpose**: Map payments to electronic invoices

**Aggregate Root**: InvoiceReference

**Key Concepts**:
- Invoice generation via Invoice Hub
- Payment-to-invoice mapping
- Retry handling

**Business Rules**:
- One invoice per payment
- Invoice number must be unique
- Failed invoice creation must be retried

## Domain Documentation Structure

Each domain follows a consistent structure:

```
domain-name/
├── overview.md           # Domain purpose and scope
├── aggregate.md          # Aggregate root and entities
├── model.md              # Value objects and types
├── business-rules.md     # Business invariants and rules
├── lifecycle.md          # State transitions and lifecycle
├── domain-events.md     # Events published by this domain
└── repositories.md       # Repository interfaces (if applicable)
```

## Key Principles

### Aggregate Boundary
- Each aggregate is a transaction boundary
- Aggregates maintain consistency
- No cross-aggregate transactions

### Business Rules
- Business rules live ONLY in domain
- Never in application, API, or infrastructure
- Encapsulated in aggregates and domain services

### Domain Events
- Events signal state changes
- Enable loose coupling
- Support event-driven architecture

## Navigation
- [Wallet Domain](./wallet/)
- [Payment Domain](./payment/)
- [Credit Transaction Domain](./credit-transaction/)
- [Ledger Domain](./ledger/)
- [Invoice Domain](./invoice/)
