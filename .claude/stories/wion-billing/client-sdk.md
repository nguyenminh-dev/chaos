# Client SDK

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** wallet-service.md, credit-service.md

---

## 1. Business Background

### Context
Client SDK là thư viện client-side để các Application (WION POS, WION FnB, AI Services) tích hợp với Billing Platform.

Mục tiêu:
- Đơn giản hóa integration
- Che đi complexity của API
- Provide type-safe interface
- Handle retry và error

### Supported Languages
- **Phase 1**: JavaScript/TypeScript (Node.js + Browser)
- **Phase 2**: Python
- **Phase 3**: Go
- **Phase 4**: Java

---

## 2. Business Concepts

### SDK Responsibilities
SDK cung cấp:
- Consume Credit
- Check Balance
- Refund Credit
- Error handling
- Retry logic
- Logging

### SDK Design Principles
- Simple: Developer chỉ cần 2-3 lines code
- Type-safe: Full TypeScript types
- Reliable: Built-in retry and error handling
- Observable: Logging và monitoring

---

## 3. Functional Requirements

### FR-10: SDK

**Actor:** Application Developer

**Precondition:**
- API credentials được cấp
- SDK được install

**Main Flow:**
1. Developer install SDK
2. Developer configure API key
3. Developer gọi SDK methods:
   - consumeCredit()
   - checkBalance()
   - refundCredit()
4. SDK handle:
   - API call
   - Retry if fail
   - Error mapping
   - Logging
5. Application nhận result

**Postcondition:**
- Integration đơn giản
- Code ngắn gọn
- Error được handle properly

---

## 4. SDK API Specification

### Initialize SDK
```typescript
import { BillingSDK } from '@wion/billing-sdk';

const sdk = new BillingSDK({
  apiKey: 'your-api-key',
  apiEndpoint: 'https://billing.wion.vn',
  tenantId: 'tenant-123',
  timeout: 5000, // optional
  retryCount: 3, // optional
});
```

### Check Balance
```typescript
const balance = await sdk.checkBalance({
  assetType: 'WI_CREDIT'
});

console.log(balance.availableBalance); // 100000
console.log(balance.reservedBalance); // 5000
```

### Consume Credit
```typescript
const result = await sdk.consumeCredit({
  amount: 1000,
  service: 'AI_GENERATION',
  referenceId: 'req-12345',
  idempotencyKey: 'unique-key-123',
  metadata: {
    resourceType: 'TOKEN',
    quantity: 1000
  }
});

console.log(result.transactionId); // txn-abc-123
console.log(result.remainingBalance); // 99000
```

### Refund Credit
```typescript
const result = await sdk.refundCredit({
  originalTransactionId: 'txn-abc-123',
  amount: 1000,
  reason: 'SERVICE_FAILURE',
  referenceId: 'refund-12345'
});

console.log(result.transactionId); // txn-refund-456
console.log(result.newBalance); // 100000
```

### Check Balance with Fallback
```typescript
try {
  const balance = await sdk.checkBalance({
    assetType: 'WI_CREDIT'
  });
  console.log('Balance:', balance.availableBalance);
} catch (error) {
  if (error instanceof InsufficientBalanceError) {
    console.error('Insufficient balance');
  } else if (error instanceof NetworkError) {
    console.error('Network error, please retry');
  } else {
    console.error('Unknown error:', error);
  }
}
```

### Advanced: Consume with Callback
```typescript
await sdk.consumeCreditWithRetry({
  amount: 1000,
  service: 'AI_GENERATION',
  referenceId: 'req-12345',
  idempotencyKey: 'unique-key-123',
  onProgress: (attempt) => {
    console.log(`Attempt ${attempt}...`);
  },
  onSuccess: (result) => {
    console.log('Success:', result.transactionId);
  },
  onFailure: (error) => {
    console.error('Failed after all retries:', error);
  }
});
```

---

## 5. Error Handling

### Error Types
```typescript
// Base error
class BillingError extends Error {
  code: string;
  details: any;
  constructor(code: string, message: string, details?: any) {
    super(message);
    this.code = code;
    this.details = details;
  }
}

// Specific errors
class InsufficientBalanceError extends BillingError {
  constructor(details: { availableBalance: number, requestedAmount: number }) {
    super('INSUFFICIENT_BALANCE', 'Not enough balance', details);
  }
}

class NetworkError extends BillingError {
  constructor(details: { url: string, statusCode?: number }) {
    super('NETWORK_ERROR', 'Network request failed', details);
  }
}

class AuthenticationError extends BillingError {
  constructor() {
    super('AUTH_ERROR', 'Invalid API key or credentials');
  }
}

class RateLimitError extends BillingError {
  constructor(details: { retryAfter: number }) {
    super('RATE_LIMIT', 'Rate limit exceeded', details);
  }
}

class IdempotencyError extends BillingError {
  constructor(details: { originalTransactionId: string }) {
    super('IDEMPOTENCY_CONFLICT', 'Request already processed', details);
  }
}
```

### Retry Strategy
```typescript
interface RetryConfig {
  maxRetries: number; // default: 3
  initialDelay: number; // default: 1000ms
  maxDelay: number; // default: 10000ms
  backoffMultiplier: number; // default: 2
  retryableErrors: string[]; // ['NETWORK_ERROR', 'TIMEOUT']
}

// Default retry behavior
- Network errors: retry
- Timeout: retry
- 5xx errors: retry
- 4xx errors: NO retry (client error)
- Idempotency conflict: NO retry (return cached result)
```

---

## 6. SDK Configuration

### Configuration Options
```typescript
interface BillingSDKConfig {
  // Required
  apiKey: string;
  apiEndpoint: string;
  tenantId: string;

  // Optional
  timeout?: number; // default: 5000ms
  retryCount?: number; // default: 3
  retryDelay?: number; // default: 1000ms
  enableLogging?: boolean; // default: true
  logLevel?: 'debug' | 'info' | 'warn' | 'error'; // default: 'info'

  // Advanced
  httpAgent?: any; // custom HTTP agent
  interceptors?: {
    request?: (config) => config;
    response?: (response) => response;
    error?: (error) => error;
  };
}
```

### Environment Configuration
```typescript
// Development
const sdk = new BillingSDK({
  apiKey: process.env.WION_BILLING_API_KEY,
  apiEndpoint: 'https://billing-dev.wion.vn',
  tenantId: process.env.WION_TENANT_ID,
  enableLogging: true,
  logLevel: 'debug'
});

// Production
const sdk = new BillingSDK({
  apiKey: process.env.WION_BILLING_API_KEY,
  apiEndpoint: 'https://billing.wion.vn',
  tenantId: process.env.WION_TENANT_ID,
  enableLogging: false
});
```

---

## 7. Logging & Monitoring

### Built-in Logging
```typescript
// SDK tự log:
// - Every API call
// - Retry attempts
// - Errors
// - Performance metrics

// Log format:
{
  timestamp: '2026-06-28T10:30:00Z',
  level: 'info',
  message: 'Billing API call',
  context: {
    method: 'consumeCredit',
    amount: 1000,
    service: 'AI_GENERATION',
    duration: 150, // ms
    attempt: 1,
    success: true
  }
}
```

### Custom Logger
```typescript
import { Logger } from '@wion/billing-sdk';

// Override default logger
Logger.setCustomLogger({
  debug: (msg, context) => console.debug('[Billing]', msg, context),
  info: (msg, context) => console.info('[Billing]', msg, context),
  warn: (msg, context) => console.warn('[Billing]', msg, context),
  error: (msg, context) => console.error('[Billing]', msg, context),
});
```

### Observability
```typescript
// Metrics (optional integration)
sdk.setMetricsCallback((metrics) => {
  // Send to your monitoring system
  console.log('Metrics:', metrics);
});

// Metrics include:
{
  apiCalls: { consumeCredit: 150, checkBalance: 300, refundCredit: 5 },
  errors: { networkError: 2, insufficientBalance: 10 },
  latency: { p50: 100, p95: 250, p99: 500 },
  retries: { total: 15, successful: 10, failed: 5 }
}
```

---

## 8. SDK Package Specification

### Package Structure
```
@wion/billing-sdk/
├── dist/
│   ├── index.js
│   ├── index.d.ts
│   └── index.min.js
├── src/
│   ├── sdk.ts
│   ├── clients/
│   │   ├── wallet-client.ts
│   │   ├── credit-client.ts
│   │   └── base-client.ts
│   ├── errors/
│   │   └── index.ts
│   ├── utils/
│   │   ├── retry.ts
│   │   ├── logger.ts
│   │   └── validator.ts
│   └── types/
│       └── index.ts
├── package.json
├── README.md
└── LICENSE
```

### Package.json
```json
{
  "name": "@wion/billing-sdk",
  "version": "1.0.0",
  "description": "WION Billing Platform SDK",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "typescript": "^5.0",
    "@types/node": "^20.0",
    "jest": "^29.0"
  },
  "keywords": ["wion", "billing", "sdk"],
  "author": "WION Platform Team",
  "license": "MIT"
}
```

---

## 9. Usage Examples

### Example 1: AI Service Integration
```typescript
import { BillingSDK } from '@wion/billing-sdk';

const sdk = new BillingSDK({
  apiKey: process.env.BILLING_API_KEY,
  apiEndpoint: process.env.BILLING_ENDPOINT,
  tenantId: process.env.TENANT_ID
});

async function generateText(prompt: string) {
  // Check balance first
  const balance = await sdk.checkBalance({ assetType: 'WI_CREDIT' });
  const estimatedCost = prompt.length * 0.1;

  if (balance.availableBalance < estimatedCost) {
    throw new Error('Insufficient balance');
  }

  // Consume credit
  const transaction = await sdk.consumeCredit({
    amount: estimatedCost,
    service: 'AI_GENERATION',
    referenceId: `gen-${Date.now()}`,
    idempotencyKey: `gen-${Date.now()}`,
    metadata: { promptLength: prompt.length }
  });

  console.log('Credit consumed:', transaction.transactionId);

  // Generate text
  const result = await aiService.generateText(prompt);

  return result;
}
```

### Example 2: Batch Processing with Idempotency
```typescript
async function processBatch(items: any[]) {
  for (const item of items) {
    try {
      const result = await sdk.consumeCredit({
        amount: item.cost,
        service: 'AI_GENERATION',
        referenceId: item.id,
        idempotencyKey: `batch-${item.id}` // ensure idempotency
      });
      console.log('Processed:', item.id);
    } catch (error) {
      if (error instanceof IdempotencyError) {
        console.log('Already processed:', item.id);
      } else {
        console.error('Failed:', item.id, error);
      }
    }
  }
}
```

### Example 3: Refund on Failure
```typescript
async function processWithRefund(itemId: string) {
  let transactionId: string;

  try {
    // Consume credit
    const result = await sdk.consumeCredit({
      amount: 1000,
      service: 'AI_GENERATION',
      referenceId: itemId,
      idempotencyKey: `process-${itemId}`
    });
    transactionId = result.transactionId;

    // Process item
    await processItem(itemId);

  } catch (error) {
    console.error('Processing failed:', error);

    // Refund credit
    if (transactionId) {
      await sdk.refundCredit({
        originalTransactionId: transactionId,
        amount: 1000,
        reason: 'PROCESSING_FAILED',
        referenceId: `refund-${itemId}`
      });
      console.log('Credit refunded');
    }

    throw error;
  }
}
```

---

## 10. Acceptance Criteria

### Definition of Done
- [ ] SDK package published
- [ ] All API methods wrapped
- [ ] Error handling hoàn chỉnh
- [ ] Retry logic hoạt động
- [ ] Logging hoạt động
- [ ] TypeScript types đầy đủ
- [ ] Documentation (README, API docs)
- [ ] Examples và guides
- [ ] Unit tests > 80%
- [ ] Integration tests với Billing API

### Test Coverage
- [ ] Test consumeCredit success
- [ ] Test consumeCredit insufficient balance
- [ ] Test consumeCredit with retry
- [ ] Test checkBalance
- [ ] Test refundCredit
- [ ] Test error mapping
- [ ] Test idempotency
- [ ] Test timeout
- [ ] Test logging

### Documentation
- [ ] Quick start guide
- [ ] API reference
- [ ] Error handling guide
- [ ] Configuration options
- [ ] Usage examples
- [ ] Migration guide (nếu có v0)

---

## 11. Cross-References

- **Depends On:**
  - wallet-service.md (wraps check balance API)
  - credit-service.md (wraps consume/refund API)

- **Dependent Services:**
  - WION POS
  - WION FnB
  - WION SPA
  - WIPIX
  - AI Services
  - Platform Services

- **Related:**
  - UC-02: Consume AI Credit
  - UC-05: Check Balance

---

## 12. Roadmap

### Phase 1 (Current)
- JavaScript/TypeScript SDK
- Core APIs: consume, refund, check balance
- Basic retry and error handling

### Phase 2
- Python SDK
- Advanced features:
  - Async/batch operations
  - Circuit breaker
  - Rate limiting

### Phase 3
- Go SDK
- gRPC support

### Phase 4
- Java SDK
- Spring Boot integration
