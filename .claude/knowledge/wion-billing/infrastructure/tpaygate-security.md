# T-PayGate Security Considerations

## Overview
Security architecture and implementation guidelines for T-PayGate integration, covering authentication, authorization, data protection, and compliance.

---

## 1. Security Layers

### 1.1 Layer 1: Transport Security

**Requirements**:
- **HTTPS Mandatory**: TLS 1.2+ required for all communications
- **Certificate Validation**: Must validate T-PayGate certificates
- **No HTTP Fallback**: Never fall back to HTTP

**Implementation**:
```csharp
public class TpgHttpClientFactory
{
    public HttpClient CreateSecureClient()
    {
        var handler = new HttpClientHandler
        {
            // Force TLS 1.2+
            SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13,
            
            // Enable certificate validation
            ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) =>
            {
                if (errors == SslPolicyErrors.None)
                    return true;
                    
                _logger.LogError("Certificate validation error: {Errors}", errors);
                return false; // Reject invalid certificates
            }
        };
        
        return new HttpClient(handler);
    }
}
```

### 1.2 Layer 2: Identity & Authentication

**OAuth 2.0 Client Credentials**:
```csharp
public class TpgOAuthCredentials
{
    // Store in secure configuration (environment variables or key vault)
    private readonly string _clientId;
    private readonly string _tenantId;
    private readonly string _source;
    
    public TpgOAuthCredentials(IConfiguration configuration)
    {
        _clientId = configuration["T-PayGate:ClientId"] 
            ?? throw new SecurityException("ClientId not configured");
        _tenantId = configuration["T-PayGate:TenantId"] 
            ?? throw new SecurityException("TenantId not configured");
        _source = configuration["T-PayGate:Source"] 
            ?? throw new SecurityException("Source not configured");
    }
    
    // Never log credentials
    public string GetClientId() => _clientId;
    public string GetTenantId() => _tenantId;
    public string GetSource() => _source;
}
```

### 1.3 Layer 3: Network Security

**IP Whitelist (Production)**:
```csharp
public class TpgIpWhitelistValidator
{
    private readonly HashSet<string> _allowedIps;
    
    public TpgIpWhitelistValidator(IConfiguration configuration)
    {
        var whitelist = configuration["T-PayGate:WebhookIpWhitelist"];
        _allowedIps = whitelist?.Split(',')?.Select(IPAddress.Parse)?.ToHashSet() 
            ?? new HashSet<IPAddress>();
    }
    
    public bool IsRequestAllowed(string remoteIpAddress)
    {
        if (_allowedIps.Count == 0)
        {
            _logger.LogWarning("IP whitelist not configured - allowing all requests");
            return true; // Staging or misconfigured
        }
        
        var ipAddress = IPAddress.Parse(remoteIpAddress);
        var isAllowed = _allowedIps.Contains(ipAddress);
        
        if (!isAllowed)
        {
            _logger.LogWarning("Webhook request from unauthorized IP: {IP}", remoteIpAddress);
        }
        
        return isAllowed;
    }
}
```

### 1.4 Layer 4: Data Integrity

**T-PayGate handles signatures internally** - per vendor specification, we do not need to implement webhook signature verification.

### 1.5 Layer 5: Credential Management

**Secret Storage**:
```csharp
public class TpgSecretManager
{
    private readonly ISecretProvider _secretProvider;
    
    public TpgSecretManager(ISecretProvider secretProvider)
    {
        _secretProvider = secretProvider;
    }
    
    // Retrieve credentials from secure storage
    public async Task<TpgCredentials> GetCredentialsAsync()
    {
        return new TpgCredentials
        {
            ClientId = await _secretProvider.GetSecretAsync("tpg-clientid"),
            TenantId = await _secretProvider.GetSecretAsync("tpg-tenantid"),
            Source = await _secretProvider.GetSecretAsync("tpg-source")
        };
    }
    
    // Credential rotation support (30-day notice)
    public async Task RotateCredentialsAsync(TpgCredentials newCredentials)
    {
        await _secretProvider.SetSecretAsync("tpg-clientid", newCredentials.ClientId);
        await _secretProvider.SetSecretAsync("tpg-tenantid", newCredentials.TenantId);
        
        _logger.LogInformation("Credentials rotated successfully");
    }
}
```

---

## 2. Credential Protection

### 2.1 Storage Requirements

**Environment Variables** (Development/Staging):
```bash
TPG_CLIENT_ID=your_client_id
TPG_TENANT_ID=your_tenant_id
TPG_SOURCE=your_source
```

**Key Vault** (Production):
```csharp
public class TpgKeyVaultSecretProvider : ISecretProvider
{
    private readonly SecretClient _secretClient;
    
    public TpgKeyVaultSecretProvider(string keyVaultUrl)
    {
        _secretClient = new SecretClient(new Uri(keyVaultUrl), 
            new DefaultAzureCredential());
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            var secret = await _secretClient.GetSecretAsync($"Tpg-{secretName}");
            return secret.Value.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError("Failed to retrieve secret {SecretName}: {Message}", 
                secretName, ex.Message);
            throw new SecurityException($"Failed to retrieve secret: {secretName}");
        }
    }
}
```

### 2.2 Logging Restrictions

**Never Log Credentials**:
```csharp
public class TpgSafeLogger
{
    public void LogApiRequest(string endpoint, object payload)
    {
        // Sanitize payload before logging
        var sanitizedPayload = SanitizePayload(payload);
        
        _logger.LogInformation("T-PayGate API call: {Endpoint} {Payload}", 
            endpoint, JsonSerializer.Serialize(sanitizedPayload));
    }
    
    private object SanitizePayload(object payload)
    {
        // Remove sensitive fields
        var json = JsonSerializer.Serialize(payload);
        var doc = JsonDocument.Parse(json);
        
        // Fields to redact
        var sensitiveFields = new[] { "clientId", "tenantId", "accessToken", "otpNumber" };
        
        foreach (var field in sensitiveFields)
        {
            if (doc.RootElement.TryGetProperty(field, out var element))
            {
                // Redact value
                // Implementation depends on JSON manipulation approach
            }
        }
        
        return payload;
    }
}
```

### 2.3 Credential Rotation

**Rotation Process**:
1. **Notice Period**: T-PayGate provides 30-day notice
2. **New Credentials**: Obtain new `clientId`/`tenantId`
3. **Transition Period**: Support both old and new credentials
4. **Old Credentials**: Revoke after transition period

**Rotation Implementation**:
```csharp
public class TpgCredentialRotator
{
    public async Task RotateCredentialsAsync(TpgCredentials oldCreds, TpgCredentials newCreds)
    {
        // Phase 1: Add new credentials alongside old ones
        await _secretProvider.SetSecretAsync("tpg-clientid-new", newCreds.ClientId);
        await _secretProvider.SetSecretAsync("tpg-tenantid-new", newCreds.TenantId);
        
        // Phase 2: Test new credentials
        try
        {
            await TestCredentialsAsync(newCreds);
            _logger.LogInformation("New credentials validated successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError("New credentials validation failed: {Message}", ex.Message);
            throw;
        }
        
        // Phase 3: Switch to new credentials
        await _secretProvider.SetSecretAsync("tpg-clientid", newCreds.ClientId);
        await _secretProvider.SetSecretAsync("tpg-tenantid", newCreds.TenantId);
        
        // Phase 4: Remove old credentials (after grace period)
        await Task.Delay(TimeSpan.FromDays(7)); // Grace period
        await _secretProvider.DeleteSecretAsync("tpg-clientid-new");
        await _secretProvider.DeleteSecretAsync("tpg-tenantid-new");
        
        _logger.LogInformation("Credential rotation completed successfully");
    }
}
```

---

## 3. Data Protection

### 3.1 PII Data Handling

**PII Fields**:
- `accountNo` - Bank account number
- `accountName` - Account holder name
- `identity` - ID card number (CMND/CCCD)
- `phone` - Phone number
- `email` - Email address

**Encryption at Rest**:
```csharp
public class TpgDataEncryptionService
{
    private readonly IEncryptionProvider _encryptionProvider;
    
    public TpgDataEncryptionService(IEncryptionProvider encryptionProvider)
    {
        _encryptionProvider = encryptionProvider;
    }
    
    public async Task<string> EncryptAccountNoAsync(string accountNo)
    {
        return await _encryptionProvider.EncryptAsync(accountNo);
    }
    
    public async Task<string> DecryptAccountNoAsync(string encryptedAccountNo)
    {
        return await _encryptionProvider.DecryptAsync(encryptedAccountNo);
    }
    
    public string MaskAccountNo(string accountNo)
    {
        // Show only last 4 digits
        return accountNo.Length > 4 
            ? $"****{accountNo.Substring(accountNo.Length - 4)}" 
            : "****";
    }
}
```

### 3.2 Data Minimization

**Collect Only Required Data**:
```csharp
public class TpgDataCollector
{
    public CollectBankConnectionDataRequest CollectMinimally(BankConnectionForm form)
    {
        // Collect only fields required by bank
        var requiredFields = GetRequiredFieldsForBank(form.BankCode);
        
        return new CollectBankConnectionDataRequest
        {
            BankCode = form.BankCode,
            AccountNo = form.AccountNo,
            AccountName = form.AccountName,
            MerchantName = form.MerchantName,
            // Only collect optional fields if required by bank
            Identity = requiredFields.Contains("identity") ? form.Identity : null,
            Phone = requiredFields.Contains("phone") ? form.Phone : null,
            Email = requiredFields.Contains("email") ? form.Email : null
        };
    }
    
    private HashSet<string> GetRequiredFieldsForBank(BankCode bankCode)
    {
        // Bank-specific field requirements (from vendor documentation)
        return bankCode.Value switch
        {
            "VCB" => new HashSet<string> { "phone" },
            "TCB" => new HashSet<string> { "identity", "phone", "email" },
            _ => new HashSet<string>()
        };
    }
}
```

### 3.3 Access Logging

**Audit Logging for PII Access**:
```csharp
public class TpgAuditLogger
{
    public void LogPiiAccess(string operation, string entityType, string entityId, 
        string userId, string[] fieldsAccessed)
    {
        var auditEvent = new AuditEvent
        {
            Timestamp = DateTimeOffset.UtcNow,
            Operation = operation,
            EntityType = entityType,
            EntityId = entityId,
            UserId = userId,
            FieldsAccessed = fieldsAccessed,
            IpAddress = GetClientIpAddress(),
            UserAgent = GetUserAgent()
        };
        
        _auditLogger.LogInformation("PII Access: {@AuditEvent}", auditEvent);
        
        // Send to audit service
        _eventBus.Publish(new PiiAccessedEvent(auditEvent));
    }
}
```

---

## 4. Webhook Security

### 4.1 IP Whitelist Validation

```csharp
[ApiController]
[Route("api/v1/webhooks/tpaygate")]
public class TpgWebhookController : ControllerBase
{
    private readonly TpgIpWhitelistValidator _ipValidator;
    
    [HttpPost]
    public async Task<IActionResult> ReceiveWebhook([FromBody] TpgWebhookPayload payload)
    {
        // Validate IP whitelist
        var remoteIp = HttpContext.Connection.RemoteIpAddress?.ToString();
        if (!_ipValidator.IsRequestAllowed(remoteIp))
        {
            _logger.LogWarning("Webhook rejected from unauthorized IP: {IP}", remoteIp);
            return StatusCode(StatusCodes.Status403Forbidden);
        }
        
        // Process webhook...
        return Ok();
    }
}
```

### 4.2 HTTPS Only

**Force HTTPS**:
```csharp
public class TpgWebhookHttpsMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.IsHttps)
        {
            _logger.LogWarning("Webhook request over HTTP - rejecting");
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            return;
        }
        
        await _next(context);
    }
}
```

### 4.3 Rate Limiting

**Webhook Rate Limiting**:
```csharp
public class TpgWebhookRateLimiter
{
    private readonly IRateLimitStore _rateLimitStore;
    
    public async Task<bool> IsAllowedAsync(string clientIp)
    {
        var key = $"webhook_limit:{clientIp}";
        var currentCount = await _rateLimitStore.GetAsync<int>(key);
        
        if (currentCount >= 100) // Max 100 requests per minute
        {
            _logger.LogWarning("Webhook rate limit exceeded for IP: {IP}", clientIp);
            return false;
        }
        
        await _rateLimitStore.IncrementAsync(key, expiresIn: TimeSpan.FromMinutes(1));
        return true;
    }
}
```

---

## 5. Input Validation

### 5.1 Request Validation

```csharp
public class TpgRequestValidator
{
    public void ValidateCreateBillRequest(CreateBillRequest request)
    {
        // Validate refTransactionId
        if (string.IsNullOrWhiteSpace(request.RefTransactionId))
            throw new ValidationException("refTransactionId is required");
            
        if (request.RefTransactionId.Length > 100)
            throw new ValidationException("refTransactionId too long (max 100 characters)");
        
        // Validate amount
        if (request.Amount <= 0)
            throw new ValidationException("Amount must be positive");
            
        if (request.Amount != Math.Floor(request.Amount))
            throw new ValidationException("Amount must be whole number (no decimals)");
        
        // Validate description
        if (!string.IsNullOrEmpty(request.Description) && request.Description.Length > 500)
            throw new ValidationException("Description too long (max 500 characters)");
    }
}
```

### 5.2 Response Validation

```csharp
public class TpgResponseValidator
{
    public void ValidateBankConnectionResponse(BankConnectionResponse response)
    {
        // Validate structure
        if (response == null)
            throw new ValidationException("Null response from T-PayGate");
        
        if (string.IsNullOrEmpty(response.ConfigBankId))
            throw new ValidationException("Missing configBankId in response");
        
        if (string.IsNullOrEmpty(response.VaNumber) && response.IsConnected)
            throw new ValidationException("Missing vaNumber for connected bank");
        
        // Validate VaNumber format (if provided)
        if (!string.IsNullOrEmpty(response.VaNumber) && !IsValidVaNumber(response.VaNumber))
            _logger.LogWarning("Unexpected VaNumber format: {VaNumber}", response.VaNumber);
    }
    
    private bool IsValidVaNumber(string vaNumber)
    {
        // Basic validation: alphanumeric, 10-20 characters
        return Regex.IsMatch(vaNumber, @"^[A-Za-z0-9]{10,20}$");
    }
}
```

---

## 6. Error Handling Security

### 6.1 Generic Error Messages

```csharp
public class TpgErrorHandler
{
    public IActionResult HandleError(Exception exception)
    {
        // Log detailed error internally
        _logger.LogError(exception, "T-PayGate API error");
        
        // Return generic error to client
        return BadRequest(new ErrorResponse
        {
            Message = "An error occurred while processing your request. Please try again.",
            Code = "INTERNAL_ERROR"
            // Do NOT expose internal details
        });
    }
}
```

### 6.2 Error Response Sanitization

```csharp
public class TpgErrorSanitizer
{
    public string SanitizeErrorMessage(string originalMessage)
    {
        // Remove potential sensitive information
        var sanitized = originalMessage
            .Replace(_credentials.ClientId, "***")
            .Replace(_credentials.TenantId, "***")
            .Replace(_credentials.Source, "***");
        
        // Remove technical details that could expose system information
        sanitized = Regex.Replace(sanitized, @"\b\d{10,}\b", "***"); // Remove long numbers
        sanitized = Regex.Replace(sanitized, @"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}", "***"); // Remove emails
        
        return sanitized;
    }
}
```

---

## 7. Compliance & Data Retention

### 7.1 GDPR Compliance

**Data Minimization**:
```csharp
public class TpgGdprComplianceService
{
    public void EnsureDataMinimization(BankConnection connection)
    {
        // Remove unnecessary data
        if (!IsRequiredByBank(connection.BankCode, "identity"))
        {
            connection.Identity = null;
        }
        
        if (!IsRequiredByBank(connection.BankCode, "email"))
        {
            connection.Email = null;
        }
    }
    
    public async Task HandleRightToErasureAsync(string tenantId)
    {
        // GDPR right to erasure (soft delete)
        await _bankConnectionRepository.SoftDeleteByTenantAsync(tenantId);
        await _billRepository.AnonymizeByTenantAsync(tenantId);
        
        _logger.LogInformation("GDPR erasure completed for tenant: {TenantId}", tenantId);
    }
}
```

### 7.2 Data Retention Policy

```csharp
public class TpgDataRetentionService
{
    public async Task ApplyRetentionPolicyAsync()
    {
        // Retention periods
        var paidBillsRetention = TimeSpan.FromDays(365 * 7); // 7 years for financial records
        var expiredBillsRetention = TimeSpan.FromDays(365);    // 1 year for analysis
        var webhookLogsRetention = TimeSpan.FromDays(90);     // 90 days for operational monitoring
        
        // Archive old data
        await ArchivePaidBillsOlderThanAsync(paidBillsRetention);
        await ArchiveExpiredBillsOlderThanAsync(expiredBillsRetention);
        await ArchiveWebhookLogsOlderThanAsync(webhookLogsRetention);
        
        _logger.LogInformation("Data retention policy applied successfully");
    }
}
```

---

## 8. Security Monitoring

### 8.1 Security Event Logging

```csharp
public class TpgSecurityEventLogger
{
    public void LogSecurityEvent(TpgSecurityEvent securityEvent)
    {
        var event = new SecurityEvent
        {
            Timestamp = DateTimeOffset.UtcNow,
            EventType = securityEvent.Type,
            Severity = securityEvent.Severity,
            Description = securityEvent.Description,
            Source = "T-PayGate",
            Details = new
            {
                IpAddress = securityEvent.IpAddress,
                UserId = securityEvent.UserId,
                Resource = securityEvent.Resource,
                Action = securityEvent.Action
            }
        };
        
        _securityLogger.LogInformation("Security Event: {@Event}", event);
        
        // Alert on critical events
        if (securityEvent.Severity == SecuritySeverity.Critical)
        {
            _alertService.SendAlertAsync(securityEvent);
        }
    }
}
```

### 8.2 Anomaly Detection

```csharp
public class TpgAnomalyDetector
{
    public void DetectAnomalies(TpgMetrics metrics)
    {
        // Detect unusual patterns
        if (metrics.ApiErrorRate > 0.05) // 5% error rate
        {
            _logger.LogWarning("Anomaly detected: High API error rate: {Rate}", metrics.ApiErrorRate);
            LogSecurityEvent(new TpgSecurityEvent
            {
                Type = "HIGH_ERROR_RATE",
                Severity = SecuritySeverity.High,
                Description = $"API error rate: {metrics.ApiErrorRate:P2}"
            });
        }
        
        if (metrics.WebhookFailureRate > 0.1) // 10% webhook failure rate
        {
            _logger.LogWarning("Anomaly detected: High webhook failure rate: {Rate}", 
                metrics.WebhookFailureRate);
            LogSecurityEvent(new TpgSecurityEvent
            {
                Type = "HIGH_WEBHOOK_FAILURE_RATE",
                Severity = SecuritySeverity.Critical,
                Description = $"Webhook failure rate: {metrics.WebhookFailureRate:P2}"
            });
        }
    }
}
```

---

## 9. Incident Response

### 9.1 Credential Exposure Response

```csharp
public class TpgIncidentResponseService
{
    public async Task HandleCredentialExposureAsync()
    {
        _logger.LogError("CRITICAL: Potential credential exposure detected");
        
        // Step 1: Alert security team immediately
        await _alertService.SendCriticalAlertAsync("T-PayGate credentials potentially exposed");
        
        // Step 2: Revoke credentials (if T-PayGate API available)
        try
        {
            await NotifyTpgCredentialExposureAsync();
            _logger.LogInformation("T-PayGate notified of credential exposure");
        }
        catch (Exception ex)
        {
            _logger.LogError("Failed to notify T-PayGate: {Message}", ex.Message);
        }
        
        // Step 3: Rotate credentials
        await InitiateEmergencyCredentialRotationAsync();
        
        // Step 4: Audit all access logs
        await AuditRecentAccessAsync();
        
        // Step 5: Implement additional monitoring
        EnableEnhancedMonitoring();
    }
}
```

---

## Related Documents
- [T-PayGate API Documentation](../../api/external/tpaygate.md) - API security requirements
- [T-PayGate Domain Overview](../../domains/payment/tpaygate/overview.md) - Domain context
- [T-PayGate Business Rules](../../domains/payment/tpaygate/business-rules.md) - Security rules
- [T-PayGate Resiliency Patterns](./tpaygate-resiliency.md) - Resilience patterns
