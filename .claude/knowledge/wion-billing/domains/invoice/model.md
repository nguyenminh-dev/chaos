# Invoice Model

## Entities

### InvoiceReference (Aggregate Root)
**Attributes**:
- `id: string` - Invoice reference ID
- `paymentId: string` - Payment ID
- `userId: string` - Tenant ID
- `invoiceNumber: string` - Invoice number
- `invoiceHubId: string` - Invoice Hub ID
- `invoiceUrl: string` - Invoice PDF URL
- `invoiceType: InvoiceType` - BAN_HANG, PHIEN_GIA_DICH, KHAC
- `status: InvoiceStatus` - PENDING, ISSUED, FAILED, CANCELLED
- `amount: decimal` - Invoice amount
- `taxAmount: decimal` - Tax amount
- `totalAmount: decimal` - Total amount (amount + tax)
- `issuedAt: datetime?` - Issue timestamp
- `createdAt: datetime` - Creation timestamp
- `updatedAt: datetime` - Update timestamp
- `metadata: json` - Additional data

**Operations**:
- `issue()` - Mark invoice as issued
- `fail(reason)` - Mark invoice as failed

## Value Objects

### InvoiceNumber
**Attributes**: `number: string`, `format: string`

**Purpose**: Invoice identifier with format validation

### InvoiceType
**Values**: `BAN_HANG`, `PHIEN_GIA_DICH`, `KHAC`

**Purpose**: Invoice classification per Vietnam regulations

### InvoiceStatus
**Values**: `PENDING`, `ISSUED`, `FAILED`, `CANCELLED`

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice Business Rules](./business-rules.md)
