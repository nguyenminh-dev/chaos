# Billing Service - API Documentation

## Overview
Billing Service provides REST APIs for wallet management, payment processing, credit operations, and invoice management.

## API Categories

### ✅ Wallet APIs - IMPLEMENTED
| API | Method | Purpose | Status |
|-----|--------|---------|--------|
| Get Wallet | `GET /api/v1/wallets/{tenantId}` | Get wallet information | ✅ Implemented |
| Get Wallet Balance | `GET /api/v1/wallets/{tenantId}/balance` | Get wallet balance details | ✅ Implemented |

### ✅ Payment APIs - IMPLEMENTED
| API | Method | Purpose | Status |
|-----|--------|---------|--------|
| Initiate Topup | `POST /api/v1/payments/topup` | Start QR code payment | ✅ Implemented |
| Get Payment Status | `GET /api/v1/payments/{paymentId}` | Get payment details | ✅ Implemented |
| Get Payment History | `GET /api/v1/payments` | Get payment history | ✅ Implemented |

### ✅ Credit APIs - IMPLEMENTED
| API | Method | Purpose | Status |
|-----|--------|---------|--------|
| Consume Credit | `POST /api/v1/credit/consume` | Consume credits | ✅ Implemented |
| Refund Credit | `POST /api/v1/credit/refund` | Refund credits | ✅ Implemented |
| Adjust Balance | `POST /api/v1/credit/adjust` | Admin balance adjustment | ✅ Implemented |
| Get Consumption History | `GET /api/v1/credit/consumption` | Get consumption history | ✅ Implemented |

### ✅ Ledger APIs - IMPLEMENTED
| API | Method | Purpose | Status |
|-----|--------|---------|--------|
| Get Transaction History | `GET /api/v1/ledger/transactions` | Get transaction history | ✅ Implemented |
| Get Transaction Detail | `GET /api/v1/ledger/transactions/{transactionId}` | Get transaction details | ✅ Implemented |

### ✅ Invoice APIs - IMPLEMENTED
| API | Method | Purpose | Status |
|-----|--------|---------|--------|
| Get Invoice by Payment | `GET /api/v1/invoices/payment/{paymentId}` | Get invoice for payment | ✅ Implemented |
| Get Invoice by Number | `GET /api/v1/invoices/{invoiceNumber}` | Get invoice details | ✅ Implemented |
| Get Invoice History | `GET /api/v1/invoices` | Get invoice history | ✅ Implemented |

## Internal APIs (Future)

### 🔄 Planned Internal APIs
| API | Method | Purpose | Called By |
|-----|--------|---------|-----------|
| Create Wallet | `POST /internal/wallets` | Create wallet | Tenant Service |
| Credit Wallet | `POST /internal/wallets/{tenantId}/credit` | Add balance | Webhook Handler |
| Payment Webhook | `POST /api/v1/webhooks/payment` | Payment callback | TPayGate |

## External APIs (Third-Party)

### 🔄 T-PayGate APIs - INTEGRATION
| API | Method | Purpose | Status |
|-----|--------|---------|--------|
| OAuth Token | `POST /oauth/token` | Get access token | Planned |
| Connect Bank | `POST /api/v1/public-api/config-bank/connect` | Connect bank account | Planned |
| Confirm OTP | `POST /api/v1/public-api/config-bank/confirm` | Verify OTP | Planned |
| List Connections | `GET /api/v1/public-api/config-bank/list` | List bank connections | Planned |
| Disconnect Bank | `POST /api/v1/public-api/config-bank/disconnect` | Disconnect bank | Planned |
| List Banks | `GET /api/v1/public-api/bank` | Get supported banks | Planned |
| Create Bill | `POST /api/v1/public-api/order/bill` | Create payment bill | Planned |
| Query Bill | `GET /api/v1/public-api/order/get-*` | Query bill status | Planned |

**Documentation**: See [T-PayGate API Documentation](./external/tpaygate.md) for complete reference

## Authentication

### Public APIs
- **Method**: API Key + Tenant ID
- **Header**: `Authorization: Bearer {api-key}`
- **Header**: `X-Tenant-Id: {tenant-id}`

### Internal APIs
- **Method**: Service-to-service authentication
- **Header**: `X-Internal-Auth: {service-token}`

### Admin APIs
- **Method**: Role-based access control
- **Roles**: Platform Admin, Finance
- **Capability**: View all tenants, manual adjustments

## Rate Limiting
- **Balance check**: 100 requests/minute per tenant
- **Credit consume**: 1,000 requests/minute per tenant
- **Payment initiation**: 10 requests/minute per tenant
- **Invoice queries**: 100 requests/minute per tenant

## Error Codes

| Code | HTTP | Description | Retryable |
|------|------|-------------|-----------|
| WALLET_NOT_FOUND | 404 | Wallet does not exist | No |
| INSUFFICIENT_BALANCE | 400 | Not enough balance | No |
| PAYMENT_NOT_FOUND | 404 | Payment does not exist | No |
| IDEMPOTENCY_CONFLICT | 409 | Duplicate idempotency key | No (return cached) |
| INVALID_SIGNATURE | 403 | Webhook signature invalid | No |
| INVOICE_FAILED | 500 | Invoice creation failed | Yes |
| INTERNAL_ERROR | 500 | Server error | Yes |

## API Versions
- **Current Version**: v1
- **Base URL**: `https://billing.wion.vn/api/v1`
- **Deprecation Policy**: 6-month notice before deprecation

## Detailed API Documentation
See [API Details](./index.md) for complete request/response examples and error handling.
