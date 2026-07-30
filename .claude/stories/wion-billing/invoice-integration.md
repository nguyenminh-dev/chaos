# Invoice Integration Service

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** payment-service.md

---

## 1. Business Background

### Context
Invoice Integration Service là cầu nối giữa Billing Platform và Invoice Hub.

Invoice Hub là hệ thống riêng biệt quản lý:
- Xuất hóa đơn điện tử
- Quản lý thuế
- Báo cáo hóa đơn
- Tra cứu hóa đơn

Billing Service KHÔNG quản lý hóa đơn điện tử.
Billing Service chỉ:
- Mapping Payment → Invoice
- Trigger invoice issuance
- Store invoice reference

---

## 2. Business Concepts

### Invoice Flow
Payment Success → Invoice Hub → Issue Invoice → Store Reference

### Timing
Invoice chỉ được tạo khi Payment thành công (BR-003).

### Mapping
Mỗi Payment có thể mapping đến:
- 1 Invoice (đơn hàng đơn)
- Nhiều Invoice (đơn hàng phức tạp, partial billing)

### Invoice Hub
External system chịu trách nhiệm:
- Kết nối với cổng hóa đơn điện tử
- Xuất hóa đơn theo luật Việt Nam
- Lưu trữ hóa đơn
- Cung cấp tra cứu và download

---

## 3. Functional Requirements

### FR-08: Xuất hóa đơn (Issue Invoice)

**Actor:** System (triggered after payment success)

**Precondition:**
- Payment thành công
- Tenant có thông tin xuất hóa đơn

**Main Flow:**
1. Webhook Handler nhận payment success
2. System trigger invoice creation
3. System collect invoice data:
   - Tenant information
   - Payment amount
   - Payment description
   - Tax information (nếu có)
4. System call Invoice Hub API
5. Invoice Hub issue invoice
6. System store invoice reference:
   - Link Payment → Invoice
   - Store invoice number
   - Store invoice URL
7. System publish InvoiceIssued event

**Postcondition:**
- Invoice được issue
- Payment có invoice reference
- Event được publish

**Alternative Flow:**
- Nếu Invoice Hub fail → Retry 3 lần
- Nếu vẫn fail → Queue for manual processing
- Alert finance team

---

## 4. Business Rules

### BR-003: Invoice Timing
Invoice chỉ được tạo khi Payment thành công.

Sequence:
1. Payment SUCCESS → Credit wallet → Create Invoice
2. Payment FAILED/TIMEOUT → Không credit, không invoice

### BR-007: Invoice Data Completeness
Invoice data phải complete trước khi issue.

Bắt buộc:
- Tenant name
- Tenant address
- Tax code (nếu là công ty)
- Payment amount
- Payment description
- Payment date

### BR-008: Invoice Retry Strategy
Invoice creation phải retry nếu fail.

Retry policy:
- Retry 3 times
- Exponential backoff: 30s, 60s, 120s
- Nếu vẫn fail → Queue manual

---

## 5. Domain Model

### InvoiceReference
```
InvoiceReference {
  id: string (PK)
  paymentId: string (FK, indexed)
  tenantId: string (indexed)
  invoiceNumber: string (indexed)
  invoiceHubId: string (indexed)
  invoiceUrl: string
  invoiceType: enum (BAN_HANG, PHIEN_GIA_DICH, KHAC)
  status: enum (PENDING, ISSUED, FAILED, CANCELLED)
  amount: decimal
  taxAmount: decimal
  totalAmount: decimal
  issuedAt: datetime
  createdAt: datetime
  updatedAt: datetime
  metadata: json
}
```

### InvoiceQueue
```
InvoiceQueue {
  id: string (PK)
  paymentId: string (indexed)
  priority: integer
  status: enum (PENDING, PROCESSING, COMPLETED, FAILED)
  retryCount: integer
  nextRetryAt: datetime
  payload: json
  errorMessage: string (nullable)
  createdAt: datetime
}
```

---

## 6. API Specification

### Create Invoice (Internal)
```
POST /api/v1/invoices/create

Request:
{
  "paymentId": "string",
  "tenantId": "string",
  "invoiceData": {
    "buyerName": "string",
    "buyerAddress": "string",
    "buyerTaxCode": "string",
    "buyerEmail": "string",
    "amount": decimal,
    "taxAmount": decimal,
    "totalAmount": decimal,
    "description": "string",
    "items": [
      {
        "name": "string",
        "quantity": integer,
        "unitPrice": decimal,
        "amount": decimal
      }
    ]
  }
}

Response:
{
  "invoiceId": "string",
  "invoiceNumber": "string",
  "invoiceUrl": "string",
  "status": "ISSUED",
  "issuedAt": "datetime"
}
```

### Get Invoice by Payment
```
GET /api/v1/invoices/payment/{paymentId}

Response:
{
  "id": "string",
  "paymentId": "string",
  "invoiceNumber": "string",
  "invoiceUrl": "string",
  "status": "ISSUED",
  "amount": decimal,
  "taxAmount": decimal,
  "totalAmount": decimal,
  "issuedAt": "datetime"
}
```

### Get Invoice by Number
```
GET /api/v1/invoices/{invoiceNumber}

Response:
{
  "id": "string",
  "invoiceNumber": "string",
  "paymentId": "string",
  "tenantId": "string",
  "status": "ISSUED",
  "amount": decimal,
  "taxAmount": decimal,
  "totalAmount": decimal,
  "issuedAt": "datetime"
}
```

### Get Invoice History
```
GET /api/v1/invoices?tenantId={tenantId}&from={from}&to={to}&status={status}&page={page}&size={size}

Response:
{
  "invoices": [
    {
      "invoiceNumber": "string",
      "amount": decimal,
      "status": "ISSUED",
      "issuedAt": "datetime"
    }
  ],
  "pagination": {...}
}
```

### Retry Invoice Creation (Internal)
```
POST /api/v1/invoices/{invoiceId}/retry

Response:
{
  "status": "QUEUED_FOR_RETRY",
  "retryAt": "datetime"
}
```

---

## 7. Integration with Invoice Hub

### Issue Invoice API
```
POST {InvoiceHub}/api/v1/invoices

Request:
{
  "buyer": {
    "name": "string",
    "address": "string",
    "taxCode": "string",
    "email": "string"
  },
  "payment": {
    "amount": decimal,
    "taxAmount": decimal,
    "totalAmount": decimal,
    "paymentDate": "datetime",
    "paymentMethod": "string"
  },
  "items": [
    {
      "name": "string",
      "quantity": integer,
      "unitPrice": decimal,
      "amount": decimal,
      "taxRate": decimal
    }
  ],
  "description": "string",
  "type": "BAN_HANG"
}

Response:
{
  "invoiceId": "string",
  "invoiceNumber": "string",
  "invoiceUrl": "string",
  "status": "ISSUED",
  "issuedAt": "datetime"
}
```

### Get Invoice Status
```
GET {InvoiceHub}/api/v1/invoices/{invoiceHubId}

Response:
{
  "invoiceId": "string",
  "invoiceNumber": "string",
  "status": "ISSUED",
  "issuedAt": "datetime",
  "pdfUrl": "string"
}
```

---

## 8. Invoice Creation Flow

### Automatic Flow
```
Payment Webhook SUCCESS
↓
Webhook Handler
↓
Credit Wallet
↓
Trigger Invoice Creation
↓
Collect Invoice Data
  - Tenant Info (name, address, tax code)
  - Payment Info (amount, date, description)
  - Tax Calculation
↓
Call Invoice Hub API
↓
Invoice Hub Issue Invoice
↓
Store Invoice Reference
  - Link payment → invoice
  - Store invoice number
↓
Publish InvoiceIssued Event
↓
Complete
```

### Retry Flow
```
Invoice Hub API FAIL
↓
Add to Invoice Queue
↓
Wait 30s
↓
Retry 1
↓
If fail → Wait 60s
↓
Retry 2
↓
If fail → Wait 120s
↓
Retry 3
↓
If still fail → Queue Manual
↓
Alert Finance Team
```

### Manual Processing Flow
```
Finance Team Review
↓
Invoice Queue Dashboard
↓
Fix Invoice Data
↓
Retry Manual
↓
Invoice Issue Success
↓
Complete
```

---

## 9. Events

### Publish
- **InvoiceIssued** - Khi invoice được issue thành công
- **InvoiceFailed** - Khi invoice creation thất bại
- **InvoiceQueuedForRetry** - Khi invoice được queue retry

### Consume
- **PaymentSucceeded** - Trigger invoice creation

---

## 10. Permission Matrix

| Actor | View Invoice | Create Invoice | Retry | View Queue |
|-------|-------------|----------------|-------|------------|
| Platform Admin | ✓ (all) | N/A | ✓ | ✓ |
| Finance | ✓ (all) | ✓ (manual) | ✓ | ✓ |
| Tenant Owner | ✓ (own) | N/A | N/A | N/A |
| System | N/A | ✓ (auto) | ✓ (auto) | N/A |
| Invoice Hub | N/A | N/A | N/A | N/A |

---

## 11. Non-functional Requirements

### Availability
- Invoice Hub integration: 99% SLA
- Queue system must be durable
- Support manual processing

### Performance
- Invoice creation async (non-blocking)
- API response < 500ms (queue acknowledgment)
- Support 500 invoices per minute

### Security
- Invoice data encrypted at rest
- Audit log for all invoice operations
- Invoice URL signed and time-limited

### Compliance
- Tuân thủ luật xuất hóa đơn điện tử Việt Nam
- Lưu trữ invoice tối thiểu 10 năm
- Support tax reporting

---

## 12. Acceptance Criteria

### Definition of Done
- [ ] Invoice creation API hoạt động
- [ ] Payment → Invoice mapping đúng
- [ ] Invoice reference được lưu
- [ ] Retry mechanism hoạt động
- [ ] Manual queue dashboard hoạt động
- [ ] Events được publish
- [ ] API documentation hoàn chỉnh
- [ ] Unit test coverage > 80%
- [ ] Integration test với Invoice Hub

### Edge Cases
- [ ] Xử lý Invoice Hub unavailable
- [ ] Xử lý invoice data incomplete
- [ ] Xử lý retry hết lần (queue manual)
- [ ] Xử lý payment không có invoice info
- [ ] Xử lý invoice issue fail (callback)
- [ ] Xử lý duplicate invoice creation
- [ ] Xử lý invoice cancellation (nếu có)

### Integration Tests
- [ ] Test invoice creation success
- [ ] Test invoice creation fail → retry → success
- [ ] Test invoice creation fail → retry → manual queue
- [ ] Test payment không có tenant info
- [ ] Test invoice Hub timeout
- [ ] Test concurrent invoice creation

---

## 13. Cross-References

- **Depends On:**
  - payment-service.md (triggered after payment success)

- **Dependent Services:**
  - ledger-service.md (log invoice operations)
  - webhook-handler.md (triggered by webhook handler)

- **Related:**
  - BR-003: Invoice timing rule
  - UC-04: Issue Invoice

---

## 14. Invoice Dashboard (Future)

Dành cho Finance Team:
- Queue management
- Retry failed invoices
- View invoice statistics
- Export invoice reports
- Bulk invoice operations
