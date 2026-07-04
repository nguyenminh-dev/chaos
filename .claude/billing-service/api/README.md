# Billing Service - API Documentation

## Overview
Billing Service provides REST APIs for wallet management, payment processing, credit operations, and invoice management.

## API Categories

### Wallet APIs
| API | Method | Purpose | Documentation |
|-----|--------|---------|----------------|
| Get Wallet Balance | `GET /api/v1/wallets/{tenantId}/balance` | Get wallet balance details | [Detailed API](./index.md#get-wallet-balance) |
| Get Wallet | `GET /api/v1/wallets/{tenantId}` | Get wallet information | [Detailed API](./index.md#get-wallet-balance) |

### Payment APIs
| API | Method | Purpose | Documentation |
|-----|--------|---------|----------------|
| Initiate Topup | `POST /api/v1/payments/topup` | Start QR code payment | [Detailed API](./index.md#initiate-topup) |
| Get Payment Status | `GET /api/v1/payments/{paymentId}` | Get payment details | [Detailed API](./index.md#get-payment-status) |
| Get Payment History | `GET /api/v1/payments` | Get payment history | [Detailed API](./index.md#get-payment-status) |

### Credit APIs
| API | Method | Purpose | Documentation |
|-----|--------|---------|----------------|
| Consume Credit | `POST /api/v1/credit/consume` | Consume credits | [Detailed API](./index.md#consume-credit) |
| Refund Credit | `POST /api/v1/credit/refund` | Refund credits | [Detailed API](./index.md#refund-credit) |
| Adjust Balance | `POST /api/v1/credit/adjust` | Admin balance adjustment | [Detailed API](./index.md#adjust-balance-admin) |
| Get Consumption History | `GET /api/v1/credit/consumption` | Get consumption history | [Detailed API](./index.md#consume-credit) |

### Ledger APIs
| API | Method | Purpose | Documentation |
|-----|--------|---------|----------------|
| Get Transaction History | `GET /api/v1/ledger/transactions` | Get transaction history | [Detailed API](./index.md#consume-credit) |
| Get Transaction Detail | `GET /api/v1/ledger/transactions/{transactionId}` | Get transaction details | [Detailed API](./index.md#consume-credit) |

### Invoice APIs
| API | Method | Purpose | Documentation |
|-----|--------|---------|----------------|
| Get Invoice by Payment | `GET /api/v1/invoices/payment/{paymentId}` | Get invoice for payment | [Detailed API](./index.md#get-invoice-by-payment) |
| Get Invoice by Number | `GET /api/v1/invoices/{invoiceNumber}` | Get invoice details | [Detailed API](./index.md#get-invoice-by-payment) |
| Get Invoice History | `GET /api/v1/invoices` | Get invoice history | [Detailed API](./index.md#get-invoice-history) |

## Internal APIs

| API | Method | Purpose | Called By |
|-----|--------|---------|-----------|
| Create Wallet | `POST /internal/wallets` | Create wallet | Tenant Service |
| Credit Wallet | `POST /internal/wallets/{tenantId}/credit` | Add balance | Webhook Handler |
| Debit Wallet | `POST /internal/wallets/{tenantId}/debit` | Deduct balance | Credit Service |
| Create Invoice | `POST /internal/invoices/create` | Create invoice | Webhook Handler |
| Payment Webhook | `POST /api/v1/webhooks/payment` | Payment callback | TPayGate |

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
