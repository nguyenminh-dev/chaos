# T-PayGate Testing Scenarios

## Overview
Comprehensive test scenarios for T-PayGate payment gateway integration covering happy paths, edge cases, failure scenarios, and non-functional requirements.

**Important Clarification**: T-PayGate is a **payment gateway provider/aggregator**, not a payment provider itself. Tests verify integration with the T-PayGate gateway, while actual payment processing is handled by banking partners through T-PayGate.

## Testing Context
- **Gateway Provider**: T-PayGate (provides unified API, bank abstraction, QR generation)
- **Payment Providers**: Banking partners (actual payment processing)
- **Our Role**: Consumer of T-PayGate gateway API
- **Integration Type**: Third-party payment gateway service integration

---

## 1. Happy Path Tests

### Test Case 1: Complete OAuth Flow
**Objective**: Verify OAuth token acquisition and usage

**Steps**:
1. Call `POST /oauth/token` with valid credentials
2. Verify response contains `accessToken` and `expiresIn: 3600`
3. Use token to call `GET /config-bank/list`
4. Verify successful response

**Expected Results**:
- Token returned with 3600 second expiry
- API call successful with valid token
- Token can be used for subsequent API calls

**Test Data**:
```json
{
  "clientId": "test_client",
  "tenantId": "test_tenant",
  "source": "TEST"
}
```

---

### Test Case 2: Bank Connection with OTP
**Objective**: Verify complete bank connection flow with OTP verification

**Steps**:
1. Call `POST /config-bank/connect` with valid bank details
2. Verify `isOTPConfirmation: true` in response
3. Save `configBankId` from response
4. Call `POST /config-bank/confirm` with correct OTP
5. Verify `isConnected: true` and `vaNumber` returned
6. Call `GET /config-bank/list` and verify connection appears

**Expected Results**:
- Connection initiated successfully
- OTP verification completes connection
- Virtual Account number allocated
- Connection appears in list with `isConnected: true`

**Test Data**:
```json
{
  "bankCode": "VCB",
  "merchantName": "Test Store",
  "accountName": "TEST USER",
  "accountNo": "1234567890",
  "phone": "0987654321"
}
```

---

### Test Case 3: Bill Creation and Payment
**Objective**: Verify complete payment flow from bill creation to payment completion

**Steps**:
1. Call `POST /order/bill` with valid `refTransactionId` and amount
2. Verify `billCode`, `qrContent`, `qrImageBase64`, `expiredAt` returned
3. Display QR code to customer
4. Simulate customer scanning QR and paying
5. Verify webhook `PAYMENT_RECEIVED` received
6. Verify webhook contains correct `RefTransactionId`, `BillCode`, `Amount`
7. Call `GET /order/get-billCode` and verify status is `PAID`

**Expected Results**:
- Bill created with QR code
- QR code scannable and valid
- Webhook received with correct payment details
- Bill status updated to PAID

**Test Data**:
```json
{
  "refTransactionId": "TEST-ORDER-001",
  "amount": 250000,
  "description": "Test payment"
}
```

---

### Test Case 4: Bill Query Operations
**Objective**: Verify bill query functionality

**Steps**:
1. Create bill with known `refTransactionId`
2. Call `GET /order/get-refTransactionId` with the ref
3. Verify bill returned in response
4. Call `GET /order/get-billCode` with the bill code
5. Verify same bill returned

**Expected Results**:
- Bill queryable by reference transaction ID
- Bill queryable by bill code
- Same bill returned via both queries

---

## 2. Edge Cases Tests

### Test Case 5: Bill Creation with Duplicate refTransactionId
**Objective**: Verify idempotency on duplicate bill creation

**Steps**:
1. Create bill with `refTransactionId: "IDEM-001"`
2. Verify bill created with `billCode: "B001"`
3. Create second bill with same `refTransactionId: "IDEM-001"`
4. Verify HTTP 409 or same bill returned (idempotency)
5. Verify no duplicate bill created in database

**Expected Results**:
- First bill creation succeeds
- Second request returns existing bill or HTTP 409
- Only one bill exists in database

---

### Test Case 6: Bill Creation with Invalid Config
**Objective**: Verify handling of invalid bank connection

**Steps**:
1. Create bill with invalid `configBankId` (via custom header)
2. Verify HTTP 404 response
3. Verify error message indicates config not found
4. Verify no bill created

**Expected Results**:
- API returns HTTP 404
- Clear error message about invalid config
- No bill created in system

---

### Test Case 7: Token Expiry Handling
**Objective**: Verify token refresh on expiry

**Steps**:
1. Use token that's about to expire (mock expiry)
2. Call API and receive HTTP 401
3. System automatically refreshes token
4. Retry API call with new token
5. Verify successful response

**Expected Results**:
- HTTP 401 returned with expired token
- Token refresh triggered automatically
- API call succeeds after token refresh

---

### Test Case 8: OTP Validation Failures
**Objective**: Verify OTP validation error handling

**Steps**:
1. Call `POST /config-bank/confirm` with wrong OTP
2. Verify HTTP 400 response
3. Call again with correct OTP
4. Verify successful connection

**Expected Results**:
- Wrong OTP returns HTTP 400
- Correct OTP after wrong OTP still works
- Connection established successfully

---

### Test Case 9: Bank Disconnection
**Objective**: Verify bank disconnection and soft delete

**Steps**:
1. Connect to bank successfully
2. Call `POST /config-bank/disconnect` with valid `configBankId`
3. Verify success response
4. Call `GET /config-bank/list`
5. Verify connection shows `isConnected: false`
6. Verify connection still exists (not deleted)

**Expected Results**:
- Disconnection succeeds
- Connection marked as inactive
- Connection record preserved (soft delete)

---

### Test Case 10: Bill Expiry
**Objective**: Verify bill expiry handling

**Steps**:
1. Create bill with expiry in the past (if configurable) or wait for expiry
2. Call `GET /order/get-billCode`
3. Verify status is `EXPIRED`

**Expected Results**:
- Bill status shows EXPIRED
- No payment accepted after expiry

---

## 3. Failure Cases Tests

### Test Case 11: Invalid OAuth Credentials
**Objective**: Verify OAuth error handling

**Steps**:
1. Call `POST /oauth/token` with invalid `clientId`
2. Verify HTTP 401 response
3. Verify error message indicates authentication failure

**Expected Results**:
- HTTP 401 returned
- Clear authentication error message
- No token generated

---

### Test Case 12: Invalid Token
**Objective**: Verify invalid token handling

**Steps**:
1. Call API with expired/invalid token
2. Verify HTTP 401 response
3. Refresh token and retry
4. Verify successful response

**Expected Results**:
- HTTP 401 with invalid token
- Token refresh resolves issue
- API call succeeds after refresh

---

### Test Case 13: API Timeout
**Objective**: Verify retry mechanism on timeout

**Steps**:
1. Mock T-PayGate API to timeout
2. Call `POST /order/bill`
3. Verify retry mechanism triggers
4. Verify circuit breaker trips after threshold
5. Verify graceful degradation

**Expected Results**:
- Retry attempts made
- Circuit breaker trips after failures
- System degrades gracefully

---

### Test Case 14: Webhook Endpoint Timeout
**Objective**: Verify webhook timeout handling

**Steps**:
1. Mock webhook endpoint to timeout (>10 seconds)
2. Simulate T-PayGate webhook delivery
3. Verify T-PayGate receives non-200 response
4. Verify T-PayGate retries webhook
5. Verify webhook eventually succeeds after endpoint recovers

**Expected Results**:
- Webhook timeout detected
- T-PayGate retries webhook
- Webhook succeeds after endpoint recovery

---

### Test Case 15: Webhook Endpoint Down
**Objective**: Verify webhook delivery failure handling

**Steps**:
1. Stop webhook endpoint
2. Simulate payment that triggers webhook
3. Verify T-PayGate retries 3 times
4. Restart webhook endpoint
5. Verify fallback polling job picks up payment
6. Verify payment processed correctly

**Expected Results**:
- All webhook retries exhausted
- Fallback polling detects payment
- Payment processed correctly via polling

---

## 4. Concurrency Tests

### Test Case 16: Concurrent Webhook Delivery
**Objective**: Verify idempotency under concurrent webhook delivery

**Steps**:
1. Send 3 identical webhooks simultaneously (same billCode)
2. Verify payment processed exactly once
3. Verify idempotency prevented double processing
4. Verify all webhooks received HTTP 200

**Expected Results**:
- Only one payment processed
- All webhook calls return HTTP 200
- Idempotency working correctly

---

### Test Case 17: Concurrent Bill Creation
**Objective**: Verify idempotency under concurrent bill creation

**Steps**:
1. Send 10 concurrent requests with same `refTransactionId`
2. Verify only 1 bill created
3. Verify all other requests receive existing bill or HTTP 409
4. Verify no race conditions in database

**Expected Results**:
- Single bill created
- All other requests handled correctly
- No database corruption

---

### Test Case 18: Concurrent Token Refresh
**Objective**: Verify token refresh thread safety

**Steps**:
1. Trigger token refresh from 10 concurrent threads
2. Verify only 1 token refresh call made to T-PayGate
3. Verify all threads receive same token
4. Verify no race conditions in cache

**Expected Results**:
- Single token refresh API call
- All threads receive same token
- Cache remains consistent

---

### Test Case 19: High-Volume Bill Creation
**Objective**: Verify system performance under load

**Steps**:
1. Create 1000 bills in rapid succession
2. Verify all bills created successfully
3. Verify no rate limiting errors
4. Verify performance within acceptable limits

**Expected Results**:
- All bills created successfully
- No rate limiting encountered
- Performance acceptable

---

## 5. Retry and Timeout Tests

### Test Case 20: API Retry on Transient Error
**Objective**: Verify retry mechanism on transient errors

**Steps**:
1. Mock T-PayGate API to return HTTP 503 once
2. Call `POST /order/bill`
3. Verify retry with exponential backoff
4. Verify request succeeds on retry

**Expected Results**:
- Initial request fails
- Retry attempts made
- Request succeeds on retry

---

### Test Case 21: Circuit Breaker Trip
**Objective**: Verify circuit breaker behavior

**Steps**:
1. Configure circuit breaker with 50% failure threshold
2. Make 10 API calls, 6 fail (60% failure rate)
3. Verify circuit breaker trips to OPEN state
4. Verify subsequent calls fail immediately
5. Wait 30 seconds
6. Verify circuit breaker transitions to HALF_OPEN
7. Make successful call
8. Verify circuit breaker closes

**Expected Results**:
- Circuit breaker trips at threshold
- Calls blocked during OPEN state
- Circuit breaker recovers after timeout
- Normal operation resumes

---

### Test Case 22: Token Refresh Retry
**Objective**: Verify token refresh retry behavior

**Steps**:
1. Mock T-PayGate OAuth to return HTTP 503 twice
2. Trigger token refresh
3. Verify retry with exponential backoff
4. Verify token refresh succeeds on 3rd attempt

**Expected Results**:
- Initial refresh attempts fail
- Retry attempts made
- Token refresh eventually succeeds

---

## 6. Idempotency Tests

### Test Case 23: Bill Creation Idempotency
**Objective**: Verify bill creation idempotency

**Steps**:
1. Create bill with `refTransactionId: "IDEM-001"`
2. Verify bill created with `billCode: "B001"`
3. Create another bill with same `refTransactionId: "IDEM-001"`
4. Verify returns same bill (`billCode: "B001"`)
5. Verify no second bill created

**Expected Results**:
- First bill creation succeeds
- Duplicate request returns existing bill
- No duplicate bill in database

---

### Test Case 24: Webhook Processing Idempotency
**Objective**: Verify webhook processing idempotency

**Steps**:
1. Deliver webhook with `billCode: "B001"`
2. Verify payment processed
3. Deliver identical webhook again
4. Verify HTTP 200 returned
5. Verify payment NOT processed again
6. Verify idempotency record exists

**Expected Results**:
- First webhook processed successfully
- Second webhook acknowledged but not processed
- Idempotency record exists

---

### Test Case 25: Bank Connection Idempotency
**Objective**: Verify bank connection idempotency

**Steps**:
1. Connect to bank with specific config
2. Receive `configBankId: "CFG-001"`
3. Attempt to connect again with identical parameters
4. Verify returns existing `configBankId: "CFG-001"`
5. Verify no duplicate connection created

**Expected Results**:
- First connection succeeds
- Duplicate connection returns existing config
- No duplicate connection in database

---

## 7. Security Tests

### Test Case 26: Credential Exposure
**Objective**: Verify credentials not exposed in logs

**Steps**:
1. Perform various API operations
2. Review application logs
3. Verify no `clientId`, `tenantId`, `accessToken` in logs
4. Verify error messages don't expose sensitive data
5. Verify webhook logs don't expose PII

**Expected Results**:
- No credentials in any logs
- No sensitive data in error messages
- PII properly protected in logs

---

### Test Case 27: Webhook Replay Attack
**Objective**: Verify webhook replay protection

**Steps**:
1. Capture webhook payload
2. Wait 1 hour
3. Re-send same webhook payload
4. Verify idempotency prevents replay
5. Verify webhook still returns HTTP 200

**Expected Results**:
- Replay detected and prevented
- Webhook acknowledged (HTTP 200)
- No duplicate payment processed

---

### Test Case 28: IP Whitelist Validation
**Objective**: Verify IP whitelist enforcement

**Steps**:
1. Send webhook from unauthorized IP
2. Verify webhook rejected (if IP whitelist implemented)
3. Send webhook from authorized IP
4. Verify webhook accepted

**Expected Results**:
- Unauthorized IP rejected
- Authorized IP accepted
- IP whitelist working correctly

---

## 8. Reconciliation Tests

### Test Case 29: Daily Reconciliation
**Objective**: Verify reconciliation process

**Steps**:
1. Create 100 bills with various statuses
2. Process 50 payments via webhooks
3. Run reconciliation job
4. Verify all payments matched
5. Verify discrepancies reported

**Expected Results**:
- All expected payments matched
- Discrepancies properly reported
- Reconciliation job completes successfully

---

### Test Case 30: Missing Webhook Detection
**Objective**: Verify fallback polling for missed webhooks

**Steps**:
1. Create bill and simulate payment
2. Block webhook delivery (endpoint down)
3. Wait for all webhook retries to exhaust
4. Run fallback polling job
5. Verify payment detected via polling
6. Verify reconciliation succeeds

**Expected Results**:
- Webhook delivery fails
- Fallback polling detects payment
- Reconciliation succeeds

---

## 9. Performance Tests

### Test Case 31: Webhook Throughput
**Objective**: Verify webhook processing performance

**Steps**:
1. Send 1000 webhooks in 1 minute
2. Verify all processed successfully
3. Verify processing time < 10 seconds per webhook
4. Verify no duplicate processing

**Expected Results**:
- All webhooks processed
- Performance within SLA
- No duplicate processing

---

### Test Case 32: API Response Time
**Objective**: Verify API response times

**Steps**:
1. Measure response time for `POST /order/bill`
2. Measure response time for `GET /config-bank/list`
3. Verify p95 response time < 2 seconds
4. Verify p99 response time < 5 seconds

**Expected Results**:
- Response times within SLA
- No significant latency spikes
- Consistent performance

---

## 10. Integration Tests

### Test Case 33: End-to-End Payment Flow
**Objective**: Verify complete payment integration

**Steps**:
1. Connect to bank (OTP flow)
2. Create bill for order
3. Display QR code to customer
4. Simulate customer payment
5. Receive webhook
6. Update order status
7. Verify order status = PAID
8. Verify reconciliation matches

**Expected Results**:
- Complete flow works end-to-end
- All components integrate correctly
- Data consistency maintained

---

### Test Case 34: Multi-Bank Scenario
**Objective**: Verify multiple bank connections

**Steps**:
1. Connect to Bank A
2. Connect to Bank B
3. Create bill for Bank A
4. Create bill for Bank B
5. Process payments for both
6. Verify payments routed correctly
7. Verify reconciliation for both banks

**Expected Results**:
- Multiple banks supported
- Payments routed correctly
- Reconciliation works for all banks

---

### Test Case 35: Error Recovery Flow
**Objective**: Verify system error recovery

**Steps**:
1. Create bill
2. Mock API failure during payment processing
3. Verify error handling
4. Verify retry mechanism
5. Verify recovery successful
6. Verify data consistency maintained

**Expected Results**:
- Errors handled gracefully
- Recovery mechanisms work
- Data consistency maintained

---

## Test Execution Guidelines

### Test Environment Setup
1. **Staging Environment**: Use T-PayGate staging for all integration tests
2. **Test Data**: Use dedicated test merchant credentials
3. **Mock Services**: Use mocks for T-PayGate unavailability testing
4. **Test Database**: Use isolated test database

### Test Execution Order
1. **Unit Tests**: Run first (fast feedback)
2. **Integration Tests**: Run after unit tests pass
3. **Contract Tests**: Run against staging environment
4. **Performance Tests**: Run during off-hours
5. **Security Tests**: Run weekly
6. **Chaos Tests**: Run monthly

### Test Data Management
1. **Isolation**: Each test should use unique data
2. **Cleanup**: Clean up test data after each test
3. **Idempotency**: Tests should be repeatable
4. **Realism**: Use realistic test data

### Test Result Reporting
1. **Pass/Fail**: Clear pass/fail criteria
2. **Metrics**: Record performance metrics
3. **Screenshots**: Capture UI states for UI tests
4. **Logs**: Preserve logs for failed tests
5. **Trends**: Track test success trends over time

---

## Related Documents
- [T-PayGate Domain Overview](../domains/payment/tpaygate/overview.md) - Domain context
- [T-PayGate Implementation Checklist](./tpaygate-implementation-checklist.md) - Implementation checklist
- [T-PayGate API Documentation](../api/external/tpaygate.md) - API reference
- [T-PayGate Resiliency Patterns](../infrastructure/tpaygate-resiliency.md) - Resilience testing
