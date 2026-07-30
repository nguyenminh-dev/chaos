# API Examples

This directory contains real-world API request/response examples for the Billing Service.

## Purpose

Examples provide:
- Ready-to-use request payloads
- Expected response structures
- Error response examples
- Integration patterns
- Testing samples

## Usage

### For Testing
Use these examples as test fixtures:
```bash
curl -X POST http://localhost:3000/api/v1/credit/consume \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test-api-key" \
  -d @examples/consume-credit-success.json
```

### For Integration
Reference these examples when integrating with the Billing Service from other applications.

### For Documentation
Copy these examples into API documentation or SDKs.

---

## Available Examples

### Wallet Operations
- [Get Wallet Balance](./wallet/get-balance.md) - Retrieve current balance
- [Get Wallet Not Found](./wallet/get-balance-not-found.md) - Error case example

### Credit Operations
- [Consume Credit Success](./credit/consume-credit-success.md) - Successful credit consumption
- [Consume Credit Insufficient Balance](./credit/consume-credit-insufficient-balance.md) - Balance validation failure
- [Refund Credit](./credit/refund-credit.md) - Credit refund operation
- [Adjust Balance](./credit/adjust-balance.md) - Admin balance adjustment

### Payment Operations
- [Initiate Topup](./payment/initiate-topup.md) - Create QR payment
- [Get Payment Status](./payment/get-payment-status.md) - Query payment state
- [Payment Webhook Success](./payment/webhook-payment-success.md) - Successful webhook callback

### Invoice Operations
- [Get Invoice by Payment](./invoice/get-invoice-by-payment.md) - Retrieve invoice
- [Get Invoice History](./invoice/get-invoice-history.md) - List invoices with pagination

---

## Example Structure

Each example file contains:

```markdown
# {Operation Name} Example

## Request
{HTTP method and endpoint}

## Request Headers
{Required headers}

## Request Body
{JSON payload}

## Response
{Expected response}

## Response Headers
{Response headers if applicable}

## Related Documentation
{Links to related docs}
```

---

## Running Examples

### Prerequisites
1. Billing Service running locally
2. Valid API key
3. Test tenant created

### Setup
```bash
# Set environment variables
export BILLING_API_URL="http://localhost:3000"
export API_KEY="your-test-api-key"
export TENANT_ID="test-tenant-123"
```

### Execute Example
```bash
# Load and execute request
curl -X POST "$BILLING_API_URL/api/v1/credit/consume" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Idempotency-Key: test-key-001" \
  -d "{
    \"tenantId\": \"$TENANT_ID\",
    \"amount\": 1000,
    \"service\": \"AI_GENERATION\",
    \"referenceId\": \"test-req-001\",
    \"idempotencyKey\": \"test-key-001\"
  }"
```

---

## Testing with Examples

### Success Path Testing
```bash
# Test successful credit consumption
./scripts/test-consume-credit-success.sh
```

### Error Path Testing
```bash
# Test insufficient balance error
./scripts/test-consume-credit-insufficient-balance.sh
```

### Load Testing
```bash
# Run load test with examples
k6 run scripts/load-test-consume-credit.js
```

---

## Example Categories

### Happy Path Examples
Demonstrate successful operations under normal conditions.

### Error Examples
Demonstrate error responses for validation failures, business rule violations, and system errors.

### Edge Cases
Demonstrate behavior for boundary conditions, rare scenarios, and special cases.

---

## Related Documentation

- [API Documentation](../api/index.md) - Complete API reference
- [Use Cases](../application/use-cases/) - Business workflows
- [Domain Knowledge](../domains/) - Business rules and constraints

---

**Last Updated**: 2026-07-05
**Maintained By**: Billing Service Team
