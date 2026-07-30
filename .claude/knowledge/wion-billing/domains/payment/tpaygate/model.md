# T-PayGate Domain Model

## Overview
Domain model following DDD principles with aggregates, value objects, and domain services. All entities follow clean architecture patterns with clear separation of concerns.

---

## Aggregates

### BankConnection Aggregate

#### Aggregate Root: BankConnection

```csharp
public class BankConnection : BillingAggregateRoot
{
    // Identity
    public Guid Id { get; private set; }
    public ConfigBankId ConfigBankId { get; private set; }
    
    // Tenant identification
    public string TenantId { get; private set; }
    
    // Bank identification
    public BankCode BankCode { get; private set; }
    public string AccountNo { get; private set; }
    public string AccountName { get; private set; }
    public string MerchantName { get; private set; }
    
    // Virtual account
    public VaNumber VaNumber { get; private set; }
    
    // Connection state
    public ConnectionStatus Status { get; private set; }
    public DateTimeOffset? ConnectedAt { get; private set; }
    public DateTimeOffset? DisconnectedAt { get; private set; }
    
    // Contact information (optional)
    public string? Phone { get; private set; }
    public string? Email { get; private set; }
    
    // Bank-specific fields (encrypted)
    public string? EncryptedPrefix { get; private set; }
    public string? EncryptedClientId { get; private set; }
    public string? EncryptedEncryptKey { get; private set; }
    public string? EncryptedSecretKey { get; private set; }
    
    // Timestamps
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset UpdatedAt { get; private set; }
    
    // Private constructor for ORM
    private BankConnection() { }
    
    // Static factory method
    public static BankConnection Initiate(
        string tenantId,
        BankCode bankCode,
        string accountNo,
        string accountName,
        string merchantName,
        string? phone = null,
        string? email = null
    )
    {
        var connection = new BankConnection
        {
            Id = Guid.NewGuid(),
            ConfigBankId = ConfigBankId.New(),
            TenantId = tenantId,
            BankCode = bankCode,
            AccountNo = accountNo,
            AccountName = accountName,
            MerchantName = merchantName,
            Status = ConnectionStatus.Initiated,
            Phone = phone,
            Email = email,
            CreatedAt = DateTimeOffset.UtcNow,
            UpdatedAt = DateTimeOffset.UtcNow
        };
        
        connection.AddDomainEvent(new BankConnectionInitiatedDomainEvent(
            connection.ConfigBankId,
            connection.TenantId,
            connection.BankCode
        ));
        
        return connection;
    }
    
    // Domain methods
    public void RequireOtp()
    {
        if (Status != ConnectionStatus.Initiated)
            throw new InvalidOperation("Cannot require OTP in current state");
            
        Status = ConnectionStatus.PendingOtp;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new BankOtpRequiredDomainEvent(ConfigBankId, BankCode));
    }
    
    public void ConfirmConnection(VaNumber vaNumber)
    {
        CheckRule(new ConnectionMustBePendingOtpRule(this));
        
        VaNumber = vaNumber;
        Status = ConnectionStatus.Connected;
        ConnectedAt = DateTimeOffset.UtcNow;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new BankConnectionConnectedDomainEvent(
            ConfigBankId,
            BankCode.Value,
            VaNumber.Value,
            AccountNo,
            MerchantName,
            ConnectedAt.Value
        ));
    }
    
    public void Disconnect(string reason = "USER_REQUESTED")
    {
        CheckRule(new ConnectionMustBeConnectedRule(this));
        
        Status = ConnectionStatus.Disconnected;
        DisconnectedAt = DateTimeOffset.UtcNow;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new BankConnectionDisconnectedDomainEvent(
            ConfigBankId,
            BankCode.Value,
            DisconnectedAt.Value,
            reason
        ));
    }
    
    public bool IsActive() => Status == ConnectionStatus.Connected;
}
```

#### Supporting Enums and Value Objects

```csharp
public enum ConnectionStatus
{
    Initiated,
    PendingOtp,
    Connected,
    Disconnected
}

public record ConfigBankId(Guid Value)
{
    public static ConfigBankId New() => new(Guid.NewGuid());
    public static ConfigBankId From(string value) => new(Guid.Parse(value));
}

public record BankCode(string Value)
{
    public static BankCode VCB => new("VCB");
    public static BankCode TCB => new("TCB");
    // Add other supported banks
}

public record VaNumber(string Value);
```

---

### Bill Aggregate

#### Aggregate Root: Bill

```csharp
public class Bill : BillingAggregateRoot
{
    // Identity
    public Guid Id { get; private set; }
    public BillCode BillCode { get; private set; }
    
    // Tenant and merchant reference
    public string TenantId { get; private set; }
    public RefTransactionId RefTransactionId { get; private set; }
    
    // Bank connection reference
    public ConfigBankId ConfigBankId { get; private set; }
    
    // Payment details
    public Amount Amount { get; private set; }
    public string? Description { get; private set; }
    
    // QR code data
    public string QrContent { get; private set; }
    public string? QrImageBase64 { get; private set; }
    
    // Bill lifecycle
    public BillStatus Status { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset ExpiredAt { get; private set; }
    public DateTimeOffset? PaidAt { get; private set; }
    public DateTimeOffset? CanceledAt { get; private set; }
    
    // Payment information
    public string? VirtualAccount { get; private set; }
    public string? ActualAccount { get; private set; }
    
    // Private constructor for ORM
    private Bill() { }
    
    // Static factory method
    public static Bill Create(
        string tenantId,
        RefTransactionId refTransactionId,
        ConfigBankId configBankId,
        Amount amount,
        string? description = null,
        DateTimeOffset? expiredAt = null
    )
    {
        var defaultExpiry = DateTimeOffset.UtcNow.AddHours(24);
        var bill = new Bill
        {
            Id = Guid.NewGuid(),
            BillCode = BillCode.New(),
            TenantId = tenantId,
            RefTransactionId = refTransactionId,
            ConfigBankId = configBankId,
            Amount = amount,
            Description = description,
            Status = BillStatus.Created,
            CreatedAt = DateTimeOffset.UtcNow,
            ExpiredAt = expiredAt ?? defaultExpiry
        };
        
        bill.AddDomainEvent(new TpgBillCreatedDomainEvent(
            bill.BillCode.Value,
            bill.RefTransactionId.Value,
            bill.TenantId,
            bill.Amount.Value
        ));
        
        return bill;
    }
    
    // Domain methods
    public void SetQrData(string qrContent, string? qrImageBase64 = null)
    {
        CheckRule(new BillMustBeInStatusRule(this, BillStatus.Created));
        
        QrContent = qrContent;
        QrImageBase64 = qrImageBase64;
        UpdatedAt = DateTimeOffset.UtcNow;
    }
    
    public void MarkAsScanned()
    {
        CheckRule(new BillMustBeInStatusRule(this, BillStatus.Created));
        
        Status = BillStatus.WaitingPayment;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgBillScannedDomainEvent(
            BillCode.Value,
            RefTransactionId.Value,
            TenantId
        ));
    }
    
    public void MarkAsPaid(string virtualAccount, string actualAccount, DateTimeOffset paymentTime)
    {
        CheckRule(new BillMustBeInStatusRule(this, BillStatus.WaitingPayment));
        CheckRule(new BillMustNotBeExpiredRule(this));
        
        Status = BillStatus.Paid;
        PaidAt = paymentTime;
        VirtualAccount = virtualAccount;
        ActualAccount = actualAccount;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgBillPaidDomainEvent(
            BillCode.Value,
            RefTransactionId.Value,
            TenantId,
            Amount.Value,
            paymentTime,
            virtualAccount,
            actualAccount
        ));
    }
    
    public void MarkAsExpired()
    {
        if (Status != BillStatus.Created && Status != BillStatus.WaitingPayment)
            return; // Already in final state
            
        Status = BillStatus.Expired;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgBillExpiredDomainEvent(
            BillCode.Value,
            RefTransactionId.Value,
            TenantId,
            ExpiredAt
        ));
    }
    
    public void Cancel()
    {
        CheckRule(new BillCanBeCanceledRule(this));
        
        Status = BillStatus.Canceled;
        CanceledAt = DateTimeOffset.UtcNow;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgBillCanceledDomainEvent(
            BillCode.Value,
            RefTransactionId.Value,
            TenantId,
            CanceledAt.Value
        ));
    }
    
    public bool IsFinalState() => Status == BillStatus.Paid || Status == BillStatus.Expired || Status == BillStatus.Canceled;
    
    public bool IsExpired() => DateTimeOffset.UtcNow > ExpiredAt;
}
```

#### Supporting Enums and Value Objects

```csharp
public enum BillStatus
{
    Created,
    WaitingPayment,
    Paid,
    Expired,
    Canceled
}

public record BillCode(string Value)
{
    public static BillCode New() => new($"B{DateTimeOffset.UtcNow:yyyyMMddHHmmss}{Random.Shared.Next(1000, 9999)}");
}

public record RefTransactionId(string Value)
{
    public static RefTransactionId From(string value) => new(value);
}

public record Amount(decimal Value)
{
    public static Amount From(decimal value)
    {
        if (value <= 0)
            throw new InvalidArgument("Amount must be positive");
        if (value != Math.Floor(value))
            throw new InvalidArgument("Amount must be whole number (no decimals)");
        return new Amount(value);
    }
}
```

---

### PaymentNotification Aggregate

#### Aggregate Root: PaymentNotification

```csharp
public class PaymentNotification : BillingAggregateRoot
{
    // Identity
    public long Id { get; private set; }
    
    // References
    public BillCode BillCode { get; private set; }
    
    // Payment details from webhook
    public Amount Amount { get; private set; }
    public string? VirtualAccount { get; private set; }
    public string? ActualAccount { get; private set; }
    public DateTimeOffset PaymentTime { get; private set; }
    
    // Processing state
    public NotificationStatus Status { get; private set; }
    public int AttemptNumber { get; private set; }
    
    // Timestamps
    public DateTimeOffset ReceivedAt { get; private set; }
    public DateTimeOffset? ProcessedAt { get; private set; }
    
    // Private constructor for ORM
    private PaymentNotification() { }
    
    // Static factory method
    public static PaymentNotification Receive(
        BillCode billCode,
        Amount amount,
        string? virtualAccount,
        string? actualAccount,
        DateTimeOffset paymentTime
    )
    {
        var notification = new PaymentNotification
        {
            BillCode = billCode,
            Amount = amount,
            VirtualAccount = virtualAccount,
            ActualAccount = actualAccount,
            PaymentTime = paymentTime,
            Status = NotificationStatus.Received,
            AttemptNumber = 1,
            ReceivedAt = DateTimeOffset.UtcNow
        };
        
        notification.AddDomainEvent(new TpgPaymentNotificationReceivedDomainEvent(
            billCode.Value,
            notification.ReceivedAt
        ));
        
        return notification;
    }
    
    // Domain methods
    public void MarkAsProcessed()
    {
        if (Status != NotificationStatus.Received && Status != NotificationStatus.Processing)
            throw new InvalidOperation("Cannot mark as processed in current state");
            
        Status = NotificationStatus.Processed;
        ProcessedAt = DateTimeOffset.UtcNow;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgPaymentNotificationProcessedDomainEvent(
            BillCode.Value,
            Amount.Value,
            ProcessedAt.Value
        ));
    }
    
    public void MarkAsDuplicate()
    {
        Status = NotificationStatus.Duplicate;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgDuplicatePaymentNotificationDetectedDomainEvent(
            BillCode.Value,
            ReceivedAt
        ));
    }
    
    public void MarkAsFailed(string failureReason)
    {
        Status = NotificationStatus.Failed;
        UpdatedAt = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new TpgPaymentNotificationFailedDomainEvent(
            BillCode.Value,
            failureReason
        ));
    }
    
    public void IncrementAttempt()
    {
        AttemptNumber++;
        UpdatedAt = DateTimeOffset.UtcNow;
    }
}
```

#### Supporting Enums

```csharp
public enum NotificationStatus
{
    Received,
    Processing,
    Processed,
    Failed,
    Duplicate
}
```

---

## Domain Services

### ITpgOAuthService

```csharp
public interface ITpgOAuthService
{
    Task<TpgAccessToken> GetAccessTokenAsync();
    Task RefreshTokenAsync();
    bool IsTokenExpired();
    bool IsTokenExpiringSoon(int minutesBefore = 5);
}

public record TpgAccessToken(string Token, int ExpiresIn, DateTimeOffset IssuedAt);
```

### ITpgBankConnectionService

```csharp
public interface ITpgBankConnectionService
{
    Task<BankConnection> InitiateConnectionAsync(InitiateConnectionCommand command);
    Task<BankConnection> ConfirmOtpAsync(ConfirmOtpCommand command);
    Task<BankConnection> DisconnectAsync(ConfigBankId configBankId);
    Task<BankConnection?> GetConnectionAsync(ConfigBankId configBankId);
    Task<List<BankConnection>> ListConnectionsAsync(string tenantId, BankCode? bankCode = null);
}
```

### ITpgBillService

```csharp
public interface ITpgBillService
{
    Task<Bill> CreateBillAsync(CreateBillCommand command);
    Task<Bill?> GetBillByRefTransactionIdAsync(RefTransactionId refTransactionId);
    Task<Bill?> GetBillByCodeAsync(BillCode billCode);
    Task<List<Bill>> GetExpiringBillsAsync();
}
```

### ITpgWebhookHandler

```csharp
public interface ITpgWebhookHandler
{
    Task HandleWebhookAsync(TpgWebhookPayload payload);
    Task<bool> IsProcessedAsync(BillCode billCode);
    Task ProcessPaymentAsync(PaymentNotification notification);
}
```

---

## Repositories

### ITpgBankConnectionRepository

```csharp
public interface ITpgBankConnectionRepository : IRepository<BankConnection, Guid>
{
    Task<BankConnection?> FindByConfigIdAsync(ConfigBankId configBankId);
    Task<BankConnection?> FindByTenantAndBankAsync(string tenantId, BankCode bankCode);
    Task<List<BankConnection>> ListActiveConnectionsAsync(string tenantId);
    Task<List<BankConnection>> ListByTenantAsync(string tenantId);
}
```

### ITpgBillRepository

```csharp
public interface ITpgBillRepository : IRepository<Bill, Guid>
{
    Task<Bill?> FindByBillCodeAsync(BillCode billCode);
    Task<Bill?> FindByRefTransactionIdAsync(RefTransactionId refTransactionId);
    Task<List<Bill>> FindActiveBillsAsync(string tenantId);
    Task<List<Bill>> FindExpiringBillsAsync(DateTimeOffset expiryThreshold);
    Task<List<Bill>> FindByStatusAsync(string tenantId, BillStatus status);
}
```

### ITpgPaymentNotificationRepository

```csharp
public interface ITpgPaymentNotificationRepository : IRepository<PaymentNotification, long>
{
    Task<PaymentNotification?> FindByBillCodeAsync(BillCode billCode);
    Task<bool> IsProcessedAsync(BillCode billCode);
    Task LogAttemptAsync(long notificationId, int attemptNumber);
    Task<List<PaymentNotification>> FindFailedNotificationsAsync(DateTimeOffset since);
}
```

---

## Specifications

```csharp
// Connection specifications
public class ActiveConnectionSpec : Specification<BankConnection>
{
    public ActiveConnectionSpec(string tenantId)
    {
        Criteria = c => c.TenantId == tenantId && c.Status == ConnectionStatus.Connected;
    }
}

// Bill specifications
public class ExpiringBillSpec : Specification<Bill>
{
    public ExpiringBillSpec(DateTimeOffset threshold)
    {
        Criteria = b => b.Status == BillStatus.Created && b.ExpiredAt <= threshold;
    }
}

public class ActiveBillSpec : Specification<Bill>
{
    public ActiveBillSpec(string tenantId)
    {
        Criteria = b => b.TenantId == tenantId && 
                     (b.Status == BillStatus.Created || b.Status == BillStatus.WaitingPayment);
    }
}
```

---

## Commands and Queries

### Commands

```csharp
// Bank Connection Commands
public record InitiateConnectionCommand(
    string TenantId,
    BankCode BankCode,
    string AccountNo,
    string AccountName,
    string MerchantName,
    string? Phone = null,
    string? Email = null
);

public record ConfirmOtpCommand(
    ConfigBankId ConfigBankId,
    string OtpNumber
);

// Bill Commands
public record CreateBillCommand(
    string TenantId,
    RefTransactionId RefTransactionId,
    ConfigBankId ConfigBankId,
    Amount Amount,
    string? Description = null
);

// Webhook Commands
public record ProcessWebhookCommand(
    string RefTransactionId,
    string BillCode,
    decimal Amount,
    string? VirtualAccount = null,
    string? ActualAccount = null,
    DateTimeOffset? PaymentTime = null
);
```

### Queries

```csharp
// Bank Connection Queries
public record GetBankConnectionQuery(ConfigBankId ConfigBankId);
public record ListBankConnectionsQuery(string TenantId, BankCode? BankCode = null);

// Bill Queries
public record GetBillByCodeQuery(BillCode BillCode);
public record GetBillByRefTransactionIdQuery(RefTransactionId RefTransactionId);
public record GetExpiringBillsQuery();
```

---

## Related Documents
- [T-PayGate Domain Overview](./overview.md) - Domain context
- [T-PayGate Aggregates](./aggregates.md) - Aggregate definitions
- [T-PayGate Business Rules](./business-rules.md) - Business rules implementation
- [T-PayGate Domain Events](./domain-events.md) - Domain events published
