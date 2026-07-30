# T-PayGate Business Rules and Constraints

## Overview
Business rules for T-PayGate payment gateway integration. **Important**: T-PayGate is a payment gateway provider that abstracts multiple banking partners. These rules govern our integration with the T-PayGate gateway, while actual payment processing is handled by banking partners through T-PayGate.

## Integration Architecture Context
- **T-PayGate**: Payment gateway provider/aggregator
- **Banking Partners**: Actual payment providers (abstracted by T-PayGate)
- **Our System**: Billing Service that integrates with T-PayGate
- **Flow**: Customer → Bank → T-PayGate → Billing Service

### Rule 1: Token Lifecycle Management
- **Token TTL**: 3600 seconds (60 minutes) exactly
- **Refresh Timing**: Must refresh at T-5 minutes (55 minutes)
- **Refresh Method**: Call `POST /api/v1/oauth/token`
- **Cache Strategy**: Store in memory with expiry timestamp
- **Fallback**: If API returns 401, refresh immediately and retry

**Violation Impact**: API access denied, payment processing failure

**Implementation**: [TpgOAuthService](../../application/tpaygate/oauth-service.md)

---

### Rule 2: Token Reuse Requirement
- **Mandatory Reuse**: Do NOT call `/token` per request
- **Rate Limiting**: `/token` endpoint is rate-limited
- **Token Scope**: Valid for all API calls (Step 2/3/4)
- **Parallel Requests**: Single token shared across all threads

**Violation Impact**: Rate limit errors, performance degradation

---

## Bank Connection Rules

### Rule 3: One-Time Bank Connection
- **Frequency**: Once per merchant-bank pair per environment
- **Output**: `configBankId` + `vaNumber` (Virtual Account)
- **Reusability**: Use same connection for all transactions
- **Duration**: Connection persists until explicitly disconnected

**Violation Impact**: Duplicate connections, payment routing failures

---

### Rule 4: OTP Requirement Variation
- **Bank Dependency**: Some banks require OTP, some don't
- **Detection**: Check `isOTPConfirmation` flag in connect response
- **OTP Validation**: Bank-specific validation rules
- **Max Retries**: 3 attempts (assumed - **clarify with vendor**)

**Violation Impact**: Connection failure, customer inconvenience

---

### Rule 5: Bank-Specific Field Requirements
- **Variable Fields**: Each bank requires different fields
- **Required vs Optional**: See bank-specific documentation
- **Validation**: Validate per bank, not generic validation
- **Common Fields**: `accountNo`, `accountName`, `merchantName` (always required)

**Violation Impact**: Connection rejection by bank

**Reference**: Bank-specific field matrix (obtain from vendor)

---

### Rule 6: Soft Delete on Disconnect
- **Deletion Method**: Set `isConnected = false`, do NOT hard delete
- **Preservation**: Keep records for reconciliation and audit
- **Reconnection**: Can reconnect after disconnect (new configBankId)
- **Payment Impact**: Disconnected configs cannot receive new payments

**Violation Impact**: Reconciliation failures, audit trail gaps

---

## Bill Creation Rules

### Rule 7: refTransactionId Uniqueness
- **Scope**: Unique per merchant (clientId + tenantId)
- **Duration**: Cached by T-PayGate for 24 hours
- **Collision**: HTTP 409 on duplicate (or returns existing bill)
- **Format**: String, alphanumeric, up to 100 characters

**Violation Impact**: Duplicate bill creation, payment confusion

**Idempotency**: Duplicate requests return existing bill

---

### Rule 8: Amount Validation
- **Currency**: VND only
- **Precision**: No decimal places (whole amounts)
- **Minimum**: Assumed 1 VND (**clarify with vendor**)
- **Maximum**: Assumed bank-specific limits (**clarify with vendor**)
- **Immutability**: Once created, amount cannot change

**Violation Impact**: Bill rejection, payment failures

---

### Rule 9: Bill Expiry Management
- **Default Expiry**: 24 hours from creation
- **Configuration**: Configurable per tenant (**clarify with vendor**)
- **Expiry Action**: Bill status → EXPIRED
- **Payment Deadline**: Payment must be completed before expiry
- **Post-Expiry**: Cannot accept payments (bank may reject)

**Violation Impact**: Invalid payment attempts, reconciliation issues

---

### Rule 10: QR Code Generation
- **Format**: Vietnam QR National standard
- **Encoding**: UTF-8
- **Rendering**: Use `qrContent` (preferred) or `qrImageBase64` (fallback)
- **Display**: Show to customer for scanning
- **Validation**: No validation required (trust T-PayGate generation)

**Violation Impact**: Payment scanning failures

---

### Rule 11: Bill Status Transitions
```
CREATED → WAITING_PAYMENT: Customer scans QR
CREATED → CANCELED: Merchant cancels before scan
CREATED → EXPIRED: 24 hours passes without scan
WAITING_PAYMENT → PAID: Payment completed
WAITING_PAYMENT → EXPIRED: Expiry time reached
```

**Prohibited Transitions**:
- PAID → any state (final state)
- EXPIRED → PAID (no payment after expiry)
- CANCELED → any state (final state)

**Violation Impact**: Data inconsistency, payment tracking errors

---

## Webhook Processing Rules

### Rule 12: Webhook Response SLA
- **Response Time**: < 10 seconds (PROD), < 30 seconds (UAT)
- **Required Response**: HTTP 200 with any body
- **Timeout Behavior**: T-PayGate retries webhook
- **Background Processing**: Queue actual processing after responding

**Violation Impact**: Webhook delivery failures, payment delays

---

### Rule 13: Idempotency by billCode
- **Idempotency Key**: `billCode` from webhook payload
- **Cache Duration**: 24 hours minimum (recommended)
- **Duplicate Detection**: Check processing status before applying payment
- **Response**: Still return HTTP 200 to acknowledge receipt

**Violation Impact**: **CRITICAL** - Double payment processing

**Implementation**: [PaymentNotification aggregate](./aggregates.md)

---

### Rule 14: Amount Validation in Webhook
- **Validation**: Webhook `Amount` MUST match `Bill.Amount`
- **Mismatch Action**: Log discrepancy, alert operations, do NOT apply payment
- **Tolerance**: Zero tolerance (exact match required)
- **Exception Handling**: Partial payments not supported

**Violation Impact**: Accounting errors, reconciliation failures

---

### Rule 15: Webhook Retry Handling
- **Retry Pattern**: T-PayGate retries 3 times × 5-minute intervals
- **Total Window**: 10 minutes from first attempt
- **Our Responsibility**: Idempotency, not retry tracking
- **Final Failure**: Alert operations team for manual intervention

**Violation Impact**: Payment notification loss

---

## Security Rules

### Rule 16: Credential Protection
- **Storage**: Environment variables or key vault, never in code
- **Logging**: Never log credentials (clientId, tenantId, accessToken)
- **Transmission**: HTTPS only (TLS 1.2+)
- **Rotation**: Support 30-day rotation notice

**Violation Impact**: **CRITICAL** - Security breach

---

### Rule 17: IP Whitelist (PROD)
- **Requirement**: Provide outbound IP addresses to T-PayGate
- **Purpose**: Webhook authentication (additional security layer)
- **Validation**: T-PayGate validates source IP
- **Updates**: Notify T-PayGate before IP changes

**Violation Impact**: Webhook rejections

---

### Rule 18: Input Validation
- **Validation**: Validate all inputs despite T-PayGate validation
- **Sanitization**: Sanitize PII data before storage
- **Length Limits**: Enforce reasonable limits on all string fields
- **Type Validation**: Strong type validation on numeric fields

**Violation Impact**: Data corruption, injection attacks

---

## Data Persistence Rules

### Rule 19: Bank Connection Retention
- **Connected Records**: Persist indefinitely
- **Disconnected Records**: Keep for reconciliation (minimum 7 years for financial data)
- **Soft Delete**: Use `isConnected = false` flag
- **Archive**: Consider archiving old disconnected records

**Violation Impact**: Reconciliation failures, compliance issues

---

### Rule 20: Bill Retention Policy
- **Paid Bills**: Keep minimum 7 years (financial records requirement)
- **Expired Bills**: Keep minimum 1 year for analysis
- **Canceled Bills**: Keep minimum 1 year for audit
- **Active Bills**: Keep until final status reached

**Violation Impact**: Compliance violations, audit failures

---

### Rule 21: Webhook Log Retention
- **Duration**: Minimum 90 days for operational monitoring
- **Content**: Request/response data, timestamps, attempt numbers
- **Purpose**: Troubleshooting, trend analysis
- **Privacy**: Sanitize PII before logging

**Violation Impact**: Operational blind spots, troubleshooting difficulties

---

## Resiliency Rules

### Rule 22: Retry Policy
- **Retryable Errors**: HTTP 5xx, network timeouts, connection errors
- **Non-Retryable**: HTTP 4xx (except 401), validation errors
- **Max Retries**: 3 attempts
- **Backoff**: Exponential with jitter (1s, 2s, 4s ± 20%)

**Violation Impact**: Cascading failures, performance degradation

---

### Rule 23: Circuit Breaker
- **Failure Threshold**: 50% error rate in 60-second window
- **Open State**: Block requests for 30 seconds
- **Half-Open**: Test with 3 requests before closing
- **Scope**: Per T-PayGate API endpoint

**Violation Impact**: System overload, extended outages

---

### Rule 24: Fallback Polling
- **Trigger**: Webhook delivery failed after all retries
- **Frequency**: Every 15 minutes
- **Duration**: Until bill reaches final state or 48 hours old
- **Action**: Query bill status, process as if webhook received

**Violation Impact**: Missed payments, customer dissatisfaction

---

## Business Logic Rules

### Rule 25: Single Payment per Bill
- **Constraint**: One bill can only be paid once
- **Partial Payments**: Not supported
- **Multiple Payments**: Create multiple bills instead
- **Validation**: Reject webhook if bill already paid

**Violation Impact**: Double payment, accounting errors

---

### Rule 26: Bank Connection Availability
- **Requirement**: Active connection required to create bills
- **Validation**: Check `isConnected = true` before bill creation
- **Fallback**: Alert merchant, suggest reconnection or alternative bank
- **Error**: Return clear error if connection inactive

**Violation Impact**: Bill creation failures

---

### Rule 27: Merchant PII Handling
- **Data Collection**: Only collect PII required by bank
- **Storage**: Encrypt at rest (account numbers, phone, email)
- **Access Logging**: Log all access to PII data
- **Retention**: Follow GDPR/data minimization principles

**Violation Impact**: Compliance violations, legal risks

---

## Reconciliation Rules

### Rule 28: Daily Reconciliation
- **Frequency**: Daily automated reconciliation
- **Source**: T-PayGate reconciliation file vs. our records
- **Action**: Report discrepancies, investigate mismatches
- **Format**: File format from vendor (**clarify with vendor**)

**Violation Impact**: Financial discrepancies, accounting errors

---

### Rule 29: Payment Timezone Handling
- **Assumption**: Timestamps in ISO 8601 format
- **Timezone**: Assumed UTC (**clarify with vendor**)
- **Display**: Convert to local timezone for customers
- **Validation**: Validate timezone on all timestamps

**Violation Impact**: Reconciliation timing errors, reporting issues

---

## Error Handling Rules

### Rule 30: Generic Error Responses
- **Principle**: Never expose sensitive data in errors
- **Format**: Generic error messages to clients
- **Logging**: Detailed errors logged for troubleshooting
- **Security**: No credentials, no internal system details

**Violation Impact**: Security vulnerabilities, information leakage

---

### Rule 31: Error Code Mapping
- **Vendor Codes**: Map T-PayGate error codes to our error codes
- **Bank Errors**: Wrapped in generic T-PayGate codes (no per-bank handling)
- **Logging**: Preserve original error codes for debugging
- **User Display**: Show user-friendly messages only

**Violation Impact**: Poor user experience, debugging difficulties

---

## Rule Violation Handling

### Detection Mechanisms
- **Validation**: Input validation at API boundaries
- **Business Rule Validation**: Domain-level rule enforcement
- **Monitoring**: Real-time monitoring of SLA violations
- **Testing**: Comprehensive test coverage for all rules

### Response Strategies
- **Critical Rules**: Block operation, alert immediately
- **High Rules**: Log warning, degrade gracefully
- **Medium Rules**: Log info, continue with fallback
- **Low Rules**: Monitor, trend analysis

### Recovery Procedures
- **Immediate**: Retry with backoff, circuit breaker reset
- **Short-term**: Manual intervention, fallback mechanisms
- **Long-term**: Process improvements, rule refinements

---

## Related Documents
- [T-PayGate Domain Overview](./overview.md) - Domain context
- [T-PayGate Aggregates](./aggregates.md) - Aggregate definitions
- [T-PayGate Domain Model](./model.md) - Class design implementing rules
- [T-PayGate Implementation Checklist](../../reference/tpaygate-implementation-checklist.md) - Rule validation checklist
