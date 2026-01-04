# 💰 Digital Wallet System - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Digital Wallet System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Wallet` | Manage account balance | Balance logic changes |
| `Transaction` | Store transaction data | Transaction model changes |
| `SpendLimit` | Define spending limits | Limit rules change |
| `FraudDetector` | Coordinate fraud checks | Detection flow changes |
| `FraudRule` | Single fraud detection rule | Specific rule logic changes |
| `WalletService` | Coordinate wallet operations | Business logic changes |

**SRP in Action:**

```java
// Wallet ONLY manages balance
public class Wallet {
    private BigDecimal balance;
    
    public void credit(BigDecimal amount) { }
    public void debit(BigDecimal amount) { }
    public boolean isWithinLimits(BigDecimal amount) { }
}

// FraudDetector ONLY checks for fraud
public class FraudDetector {
    private final List<FraudRule> rules;
    
    public List<String> check(Transaction txn, Wallet wallet) { }
}

// WalletService coordinates operations
public class WalletService {
    public Transaction transfer(String from, String to, BigDecimal amount) {
        // Validate → Check fraud → Execute → Record
    }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Fraud Rules:**

```java
// No changes to existing code needed
public class LocationRule implements FraudRule {
    @Override
    public boolean isSuspicious(Transaction txn, Wallet wallet) {
        // Check if transaction from unusual location
    }
}

public class DeviceRule implements FraudRule {
    @Override
    public boolean isSuspicious(Transaction txn, Wallet wallet) {
        // Check if transaction from new device
    }
}

// Just add to detector
fraudDetector.addRule(new LocationRule());
fraudDetector.addRule(new DeviceRule());
```

**Adding New Transaction Types:**

```java
public enum TransactionType {
    DEPOSIT,
    WITHDRAWAL,
    TRANSFER_OUT,
    TRANSFER_IN,
    REFUND,
    CASHBACK,        // New!
    REWARD_POINTS    // New!
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All fraud rules work interchangeably:**

```java
public class FraudDetector {
    private final List<FraudRule> rules;
    
    public List<String> check(Transaction txn, Wallet wallet) {
        List<String> reasons = new ArrayList<>();
        
        // Any FraudRule implementation works here
        for (FraudRule rule : rules) {
            if (rule.isSuspicious(txn, wallet)) {
                reasons.add(rule.getReason());
            }
        }
        
        return reasons;
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
public interface FraudRule {
    boolean isSuspicious(Transaction transaction, Wallet wallet);
    String getReason();
}
```

**Could be extended:**

```java
public interface TransactionValidator {
    boolean validate(Transaction transaction);
    String getValidationError();
}

public interface WalletValidator {
    boolean validate(Wallet wallet);
    String getValidationError();
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
public class WalletService {
    private final FraudDetector fraudDetector;  // Concrete class
}
```

**Better with DIP:**

```java
public interface FraudDetectionService {
    List<String> check(Transaction txn, Wallet wallet);
    boolean isSuspicious(Transaction txn, Wallet wallet);
}

public interface WalletRepository {
    void save(Wallet wallet);
    Wallet findById(String id);
    List<Wallet> findByUserId(String userId);
}

public class WalletService {
    private final FraudDetectionService fraudService;
    private final WalletRepository walletRepo;
    
    public WalletService(FraudDetectionService fraud, WalletRepository repo) {
        this.fraudService = fraud;
        this.walletRepo = repo;
    }
}
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Wallet manages balance, Transaction stores transaction data, FraudDetector checks fraud, WalletService coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new fraud rules) without modifying existing code. Strategy pattern with FraudRule interface enables this. | N/A | - |
| **LSP** | PASS | All FraudRule implementations properly implement the FraudRule interface contract. They are substitutable. | N/A | - |
| **ISP** | PASS | FraudRule interface is minimal and focused. Clients only implement what they need. No unused methods. | N/A | - |
| **DIP** | WEAK | WalletService depends on concrete FraudDetector. Could depend on FraudDetectionService interface. Mentioned in DIP section but not fully implemented. | Extract FraudDetectionService and WalletRepository interfaces | More abstraction layers, but improves testability and flexibility |

---

## Design Patterns Used

### 1. Builder Pattern

**Where:** Transaction creation

```java
Transaction transaction = new Transaction.Builder()
    .type(TransactionType.TRANSFER_OUT)
    .amount(new BigDecimal("100.00"))
    .currency(Currency.USD)
    .from(fromWalletId)
    .to(toWalletId)
    .description("Payment for services")
    .build();
```

**Benefits:**
- Clear, readable transaction creation
- Optional fields without constructor overloading
- Immutable transactions after creation

---

### 2. Strategy Pattern

**Where:** Fraud detection rules

```java
public interface FraudRule {
    boolean isSuspicious(Transaction transaction, Wallet wallet);
}

public class LargeAmountRule implements FraudRule { }
public class FrequencyRule implements FraudRule { }
public class VelocityRule implements FraudRule { }
```

**Benefits:**
- Easy to add new fraud rules
- Rules can be combined dynamically
- Each rule is independently testable

---

### 3. Template Method Pattern (Potential)

**Where:** Transaction processing flow

```java
public abstract class TransactionProcessor {
    
    public final Transaction process(TransactionRequest request) {
        validate(request);
        checkFraud(request);
        Transaction txn = execute(request);
        record(txn);
        notify(txn);
        return txn;
    }
    
    protected abstract void validate(TransactionRequest request);
    protected abstract Transaction execute(TransactionRequest request);
    
    protected void checkFraud(TransactionRequest request) { /* default */ }
    protected void record(Transaction txn) { /* default */ }
    protected void notify(Transaction txn) { /* default */ }
}

public class DepositProcessor extends TransactionProcessor { }
public class WithdrawalProcessor extends TransactionProcessor { }
public class TransferProcessor extends TransactionProcessor { }
```

---

### 4. Observer Pattern (Potential)

**Where:** Transaction notifications

```java
public interface TransactionObserver {
    void onTransactionCompleted(Transaction transaction);
    void onTransactionFlagged(Transaction transaction, List<String> reasons);
}

public class NotificationService implements TransactionObserver {
    @Override
    public void onTransactionCompleted(Transaction txn) {
        sendNotification(txn.getFromWalletId(), "Transaction completed");
    }
}

public class AuditService implements TransactionObserver {
    @Override
    public void onTransactionFlagged(Transaction txn, List<String> reasons) {
        logForReview(txn, reasons);
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single Service Class with All Logic

**What it is:**

```java
// Rejected approach
public class WalletService {
    private Map<String, BigDecimal> balances;
    private Map<String, List<Transaction>> transactions;
    
    public void transfer(String from, String to, BigDecimal amount) {
        // All logic here: validation, fraud check, balance update, logging
        if (balances.get(from) < amount) {
            throw new IllegalStateException("Insufficient balance");
        }
        // Fraud check logic mixed in
        if (amount > 5000) {
            // Flag transaction
        }
        balances.put(from, balances.get(from).subtract(amount));
        balances.put(to, balances.get(to).add(amount));
        // Log transaction
    }
}
```

**Why rejected:**
- Violates Single Responsibility Principle (SRP) - service handles validation, fraud detection, balance management, and logging
- Hard to test individual components in isolation
- Difficult to extend with new fraud rules or limit types
- Fraud detection logic mixed with transaction logic makes code harder to maintain

**What breaks:**
- Adding new fraud rules requires modifying WalletService
- Cannot reuse fraud detection logic in other contexts
- Testing fraud rules independently requires mocking entire service
- Changes to fraud logic risk breaking transaction logic

---

### Alternative 2: Database for Transaction History

**What it is:**

```java
// Rejected approach
public class WalletService {
    private Database db;
    
    public void transfer(String from, String to, BigDecimal amount) {
        db.execute("UPDATE wallets SET balance = balance - ? WHERE id = ?", amount, from);
        db.execute("UPDATE wallets SET balance = balance + ? WHERE id = ?", amount, to);
        db.execute("INSERT INTO transactions (from_id, to_id, amount) VALUES (?, ?, ?)", 
                   from, to, amount);
    }
    
    public List<Transaction> getHistory(String walletId) {
        return db.query("SELECT * FROM transactions WHERE from_id = ? OR to_id = ?", 
                       walletId, walletId);
    }
}
```

**Why rejected:**
- Overkill for in-memory system - adds unnecessary complexity
- Database operations add latency for simple queries
- Requires database setup and connection management
- Violates requirement for simple, interview-friendly design
- Harder to demonstrate concurrency control with database transactions

**What breaks:**
- Simple operations become complex with database overhead
- Cannot easily demonstrate thread-safe operations with in-memory structures
- Requires external dependencies (database driver, connection pool)
- Interview setting assumes in-memory implementation

---

## Concurrency Handling

### Wallet Locking

```java
public class Wallet {
    private final ReentrantLock lock;
    
    public void credit(BigDecimal amount) {
        lock.lock();
        try {
            // Modify balance
            this.balance = this.balance.add(amount);
        } finally {
            lock.unlock();
        }
    }
    
    public void debit(BigDecimal amount) {
        lock.lock();
        try {
            if (balance.compareTo(amount) < 0) {
                throw new IllegalStateException("Insufficient balance");
            }
            this.balance = this.balance.subtract(amount);
        } finally {
            lock.unlock();
        }
    }
}
```

### Transfer Atomicity

```java
public Transaction transfer(String fromId, String toId, BigDecimal amount) {
    // Lock both wallets in consistent order to prevent deadlock
    synchronized (this) {
        fromWallet.debit(amount);
        toWallet.credit(amount);
    }
}
```

**Deadlock Prevention:**
- Use global lock for transfers
- Alternative: Lock wallets in consistent order (by ID)

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle idempotency?

**Answer:**

```java
public class IdempotentWalletService {
    private final Map<String, Transaction> processedRequests;
    
    public Transaction transfer(String idempotencyKey, String from, 
                               String to, BigDecimal amount) {
        // Check if already processed
        Transaction existing = processedRequests.get(idempotencyKey);
        if (existing != null) {
            return existing;  // Return same result
        }
        
        // Process new request
        Transaction result = doTransfer(from, to, amount);
        
        // Store for future duplicate requests
        processedRequests.put(idempotencyKey, result);
        
        return result;
    }
}
```

---

### Q2: How would you implement transaction rollback?

**Answer:**

```java
public Transaction transferWithRollback(String from, String to, 
                                        BigDecimal amount) {
    Wallet fromWallet = getWallet(from);
    Wallet toWallet = getWallet(to);
    
    try {
        fromWallet.debit(amount);
        
        try {
            toWallet.credit(amount);
        } catch (Exception e) {
            // Rollback debit
            fromWallet.credit(amount);
            throw e;
        }
        
        return createCompletedTransaction();
        
    } catch (Exception e) {
        return createFailedTransaction(e.getMessage());
    }
}
```

---

### Q3: How would you implement audit logging?

**Answer:**

```java
public class AuditLog {
    private final String action;
    private final String userId;
    private final String walletId;
    private final String details;
    private final LocalDateTime timestamp;
    private final String ipAddress;
}

public class AuditService {
    public void log(String action, String userId, String walletId, 
                   String details) {
        AuditLog entry = new AuditLog(action, userId, walletId, 
            details, LocalDateTime.now(), getCurrentIP());
        auditRepository.save(entry);
    }
}

// Usage in WalletService
public Transaction withdraw(String walletId, BigDecimal amount, 
                           String destination) {
    auditService.log("WITHDRAWAL_ATTEMPT", getUserId(), walletId,
        "Amount: " + amount + ", Destination: " + destination);
    
    Transaction result = doWithdraw(walletId, amount, destination);
    
    auditService.log("WITHDRAWAL_" + result.getStatus(), getUserId(), 
        walletId, "Transaction: " + result.getId());
    
    return result;
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add KYC verification** - Identity verification before large transactions
2. **Add two-factor authentication** - For sensitive operations
3. **Add notification system** - Email/SMS for transactions
4. **Add statement generation** - Monthly statements with PDF export
5. **Add interest calculation** - For savings wallets
6. **Add merchant integration** - For payment processing
7. **Add multi-currency support** - Support multiple currencies
8. **Add transaction scheduling** - Schedule recurring payments

---

### Q5: How would you handle distributed wallet operations?

**Answer:**

```java
public class DistributedWalletService {
    private final DistributedLock lockService;  // Redis/Zookeeper
    private final WalletRepository walletRepo;
    
    public Transaction transfer(String fromId, String toId, BigDecimal amount) {
        // Acquire distributed locks in consistent order (by wallet ID)
        String lock1 = fromId.compareTo(toId) < 0 ? fromId : toId;
        String lock2 = fromId.compareTo(toId) < 0 ? toId : fromId;
        
        Lock lock = lockService.acquireLock(lock1 + ":" + lock2);
        try {
            Wallet from = walletRepo.findById(fromId);
            Wallet to = walletRepo.findById(toId);
            
            from.debit(amount);
            to.credit(amount);
            
            walletRepo.save(from);
            walletRepo.save(to);
            
            return createCompletedTransaction();
        } finally {
            lock.release();
        }
    }
}
```

**Considerations:**
- Use consistent locking order to prevent deadlocks
- Distributed locks add network latency
- Need lock timeout to prevent indefinite blocking
- Consider optimistic locking for better performance

---

### Q6: How would you implement transaction batching?

**Answer:**

```java
public class BatchedWalletService {
    private final Queue<TransactionRequest> batchQueue;
    private final ScheduledExecutorService scheduler;
    
    public void transferAsync(String fromId, String toId, BigDecimal amount) {
        TransactionRequest request = new TransactionRequest(fromId, toId, amount);
        batchQueue.offer(request);
    }
    
    @PostConstruct
    public void startBatchProcessor() {
        scheduler.scheduleAtFixedRate(() -> {
            List<TransactionRequest> batch = new ArrayList<>();
            batchQueue.drainTo(batch, 100);  // Batch up to 100 transactions
            
            if (!batch.isEmpty()) {
                processBatch(batch);
            }
        }, 0, 100, TimeUnit.MILLISECONDS);
    }
    
    private void processBatch(List<TransactionRequest> batch) {
        // Group by wallet ID to minimize lock contention
        Map<String, List<TransactionRequest>> grouped = batch.stream()
            .collect(Collectors.groupingBy(TransactionRequest::getFromWalletId));
        
        for (Map.Entry<String, List<TransactionRequest>> entry : grouped.entrySet()) {
            processWalletBatch(entry.getKey(), entry.getValue());
        }
    }
}
```

**Benefits:**
- Reduces lock contention by grouping operations
- Improves throughput for high-volume scenarios
- Can optimize database writes

**Tradeoffs:**
- Adds latency (transactions not immediate)
- More complex error handling
- Need idempotency for retries

---

### Q7: How would you implement wallet sharding?

**Answer:**

```java
public class ShardedWalletService {
    private final List<WalletService> shards;
    private final int shardCount;
    
    public ShardedWalletService(int shardCount) {
        this.shardCount = shardCount;
        this.shards = new ArrayList<>();
        for (int i = 0; i < shardCount; i++) {
            shards.add(new WalletService());
        }
    }
    
    private WalletService getShard(String walletId) {
        int shardIndex = Math.abs(walletId.hashCode()) % shardCount;
        return shards.get(shardIndex);
    }
    
    public Transaction transfer(String fromId, String toId, BigDecimal amount) {
        WalletService fromShard = getShard(fromId);
        WalletService toShard = getShard(toId);
        
        if (fromShard == toShard) {
            // Same shard - simple transfer
            return fromShard.transfer(fromId, toId, amount);
        } else {
            // Cross-shard transfer - need distributed transaction
            return transferCrossShard(fromShard, toShard, fromId, toId, amount);
        }
    }
    
    private Transaction transferCrossShard(WalletService fromShard,
                                          WalletService toShard,
                                          String fromId, String toId,
                                          BigDecimal amount) {
        // Two-phase commit or saga pattern
        Transaction txn = fromShard.beginTransfer(fromId, toId, amount);
        try {
            toShard.completeTransfer(txn);
            fromShard.commitTransfer(txn);
            return txn;
        } catch (Exception e) {
            fromShard.rollbackTransfer(txn);
            throw e;
        }
    }
}
```

**Benefits:**
- Distributes load across multiple services
- Scales horizontally
- Reduces lock contention per shard

**Tradeoffs:**
- Cross-shard transfers require distributed transactions
- More complex failure handling
- Need consistent hashing for rebalancing

---

### Q8: How would you optimize fraud detection for high throughput?

**Answer:**

```java
public class OptimizedFraudDetector {
    private final FraudDetector detector;
    private final Cache<String, FraudCheckResult> cache;  // Cache recent checks
    private final AsyncFraudProcessor asyncProcessor;
    
    public List<String> check(Transaction transaction, Wallet wallet) {
        // Quick synchronous checks first
        List<String> quickChecks = performQuickChecks(transaction, wallet);
        if (!quickChecks.isEmpty()) {
            return quickChecks;
        }
        
        // Check cache for recent similar transactions
        String cacheKey = generateCacheKey(transaction, wallet);
        FraudCheckResult cached = cache.get(cacheKey);
        if (cached != null && !cached.isExpired()) {
            return cached.getReasons();
        }
        
        // Perform full fraud check
        List<String> reasons = detector.check(transaction, wallet);
        
        // Cache result for future similar transactions
        cache.put(cacheKey, new FraudCheckResult(reasons, Duration.ofMinutes(5)));
        
        // Async deep analysis for flagged transactions
        if (!reasons.isEmpty()) {
            asyncProcessor.processAsync(transaction, wallet, reasons);
        }
        
        return reasons;
    }
    
    private List<String> performQuickChecks(Transaction txn, Wallet wallet) {
        List<String> reasons = new ArrayList<>();
        
        // Rule 1: Amount threshold (O(1))
        if (txn.getAmount().compareTo(new BigDecimal("5000")) > 0) {
            reasons.add("Large amount");
        }
        
        // Rule 2: Wallet frozen (O(1))
        if (wallet.isFrozen()) {
            reasons.add("Wallet frozen");
        }
        
        return reasons;
    }
}
```

**Optimizations:**
- Caching reduces redundant fraud checks
- Quick synchronous checks for fast rejection
- Async processing for complex rules
- Rule ordering (fastest rules first)

**Tradeoffs:**
- Cache invalidation complexity
- Eventual consistency for async checks
- More complex code

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `deposit` | O(1) | Direct balance update |
| `withdraw` | O(h) | h = history size for limit check |
| `transfer` | O(h) | h = history size for fraud check |
| `getBalance` | O(1) | Direct field access |
| `getHistory` | O(n) | n = transaction count |

### Space Complexity

| Component | Space |
|-----------|-------|
| Wallets | O(w) | w = number of wallets |
| Transactions | O(t) | t = total transactions |
| History per wallet | O(h) | h = wallet transactions |

### Bottlenecks at Scale

**10x Usage (10K → 100K wallets, 1K → 10K transactions/second):**
- Problem: Per-wallet lock contention increases, transaction history storage grows, fraud detection becomes expensive
- Solution: Use distributed locking (Redis) for wallet operations, implement transaction archiving, optimize fraud detection algorithms
- Tradeoff: Additional infrastructure (Redis), network latency for distributed operations

**100x Usage (10K → 1M wallets, 1K → 100K transactions/second):**
- Problem: Single instance can't handle all wallets, database becomes bottleneck, transaction processing too slow
- Solution: Shard wallets by user ID hash, use distributed transaction coordinator, implement event-driven architecture for fraud detection
- Tradeoff: Distributed system complexity, need transaction coordination and eventual consistency handling


### Q1: How would you handle multi-currency transfers?

```java
public interface CurrencyConverter {
    BigDecimal convert(BigDecimal amount, Currency from, Currency to);
    BigDecimal getExchangeRate(Currency from, Currency to);
}

public class WalletService {
    private final CurrencyConverter converter;
    
    public Transaction transferWithConversion(String fromId, String toId, 
                                              BigDecimal amount) {
        Wallet from = getWallet(fromId);
        Wallet to = getWallet(toId);
        
        BigDecimal convertedAmount = converter.convert(
            amount, from.getCurrency(), to.getCurrency());
        
        from.debit(amount);
        to.credit(convertedAmount);
        
        // Record both amounts in transaction
        return createTransaction(amount, convertedAmount, 
            from.getCurrency(), to.getCurrency());
    }
}
```

### Q2: How would you implement scheduled payments?

```java
public class ScheduledPayment {
    private final String id;
    private final String fromWalletId;
    private final String toWalletId;
    private final BigDecimal amount;
    private final RecurrencePattern pattern;
    private LocalDateTime nextExecution;
    private boolean active;
}

public class ScheduledPaymentService {
    private final ScheduledExecutorService scheduler;
    
    public void schedulePayment(ScheduledPayment payment) {
        scheduler.scheduleAtFixedRate(
            () -> executePayment(payment),
            calculateInitialDelay(payment),
            payment.getPattern().getIntervalMs(),
            TimeUnit.MILLISECONDS);
    }
    
    private void executePayment(ScheduledPayment payment) {
        try {
            walletService.transfer(
                payment.getFromWalletId(),
                payment.getToWalletId(),
                payment.getAmount());
        } catch (Exception e) {
            handleFailedPayment(payment, e);
        }
    }
}
```

### Q3: How would you implement refunds?

```java
public Transaction refund(String originalTransactionId, BigDecimal amount) {
    Transaction original = getTransaction(originalTransactionId);
    
    if (original.getStatus() != TransactionStatus.COMPLETED) {
        throw new IllegalStateException("Can only refund completed transactions");
    }
    
    if (amount.compareTo(original.getAmount()) > 0) {
        throw new IllegalArgumentException("Refund exceeds original amount");
    }
    
    // Reverse the transaction
    Transaction refund = new Transaction.Builder()
        .type(TransactionType.REFUND)
        .amount(amount)
        .from(original.getToWalletId())
        .to(original.getFromWalletId())
        .description("Refund for " + originalTransactionId)
        .build();
    
    // Execute refund (no fraud check for refunds)
    getWallet(original.getToWalletId()).debit(amount);
    getWallet(original.getFromWalletId()).credit(amount);
    
    refund.complete();
    return refund;
}
```

### Q4: How would you implement transaction disputes?

```java
public class Dispute {
    private final String id;
    private final String transactionId;
    private final String reason;
    private DisputeStatus status;
    private final LocalDateTime createdAt;
    private String resolution;
}

public class DisputeService {
    public Dispute createDispute(String transactionId, String reason) {
        Transaction txn = walletService.getTransaction(transactionId);
        
        // Temporarily hold funds if needed
        if (txn.getType() == TransactionType.TRANSFER_OUT) {
            holdFunds(txn.getToWalletId(), txn.getAmount());
        }
        
        return new Dispute(transactionId, reason);
    }
    
    public void resolveDispute(String disputeId, boolean favorCustomer) {
        Dispute dispute = getDispute(disputeId);
        Transaction txn = walletService.getTransaction(dispute.getTransactionId());
        
        if (favorCustomer) {
            // Issue refund
            walletService.refund(txn.getId(), txn.getAmount());
        }
        
        releaseHold(txn.getToWalletId());
        dispute.resolve(favorCustomer ? "Refunded" : "Denied");
    }
}
```

