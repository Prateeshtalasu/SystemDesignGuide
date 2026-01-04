# Payment System - Production Deep Dives (Core)

## Overview

This document covers the core production components: idempotency implementation, double-entry bookkeeping, fraud detection, and payment processing flows.

---

## 1. Idempotency

### A) CONCEPT: What is Idempotency in Payment Context?

Idempotency ensures that processing the same payment request multiple times produces the same result. This is critical for payment systems because:

1. **Network Retries**: Clients may retry failed requests
2. **Timeout Handling**: Requests may timeout but succeed on server
3. **Duplicate Prevention**: Prevents charging customers twice
4. **Compliance**: Financial regulations require accurate transaction records

**Example:**
```
Client sends payment request with idempotency_key="abc123"
  ↓
Server processes payment, charges $100
  ↓
Network timeout, client doesn't receive response
  ↓
Client retries with same idempotency_key="abc123"
  ↓
Server recognizes key, returns cached result (no duplicate charge)
```

### B) OUR USAGE: How We Implement Idempotency

**Storage:**
- **Redis**: Fast lookups for idempotency keys
- **PostgreSQL**: Persistent storage for audit trail
- **TTL**: 24 hours (covers retry window)

**Key Structure:**
```
Redis Key: idempotency:{idempotency_key}
Value: Serialized PaymentResult
TTL: 24 hours
```

**Implementation:**

```java
@Service
public class IdempotencyService {
    
    private final RedisTemplate<String, PaymentResult> redis;
    private final TransactionRepository transactionRepository;
    private static final Duration TTL = Duration.ofHours(24);
    
    public PaymentResult processPayment(PaymentRequest request) {
        String idempotencyKey = request.getIdempotencyKey();
        
        if (idempotencyKey == null || idempotencyKey.isEmpty()) {
            throw new InvalidRequestException("Idempotency-Key header required");
        }
        
        // 1. Check Redis cache
        String cacheKey = "idempotency:" + idempotencyKey;
        PaymentResult cached = redis.opsForValue().get(cacheKey);
        
        if (cached != null) {
            // Return cached result (same status code + body)
            return cached;
        }
        
        // 2. Check database (for persistence across Redis restarts)
        Transaction existing = transactionRepository.findByIdempotencyKey(idempotencyKey);
        if (existing != null) {
            PaymentResult result = PaymentResult.from(existing);
            // Re-cache in Redis
            redis.opsForValue().set(cacheKey, result, TTL);
            return result;
        }
        
        // 3. Process payment (new request)
        PaymentResult result = processNewPayment(request, idempotencyKey);
        
        // 4. Cache result
        redis.opsForValue().set(cacheKey, result, TTL);
        
        return result;
    }
    
    private PaymentResult processNewPayment(PaymentRequest request, String idempotencyKey) {
        // Use distributed lock to prevent race conditions
        String lockKey = "lock:idempotency:" + idempotencyKey;
        
        if (distributedLock.tryLock(lockKey, Duration.ofSeconds(5))) {
            try {
                // Double-check after acquiring lock
                PaymentResult cached = redis.opsForValue().get("idempotency:" + idempotencyKey);
                if (cached != null) {
                    return cached;
                }
                
                // Process payment
                PaymentResult result = paymentProcessor.charge(request);
                
                // Store in database
                Transaction transaction = Transaction.builder()
                    .transactionId(result.getPaymentId())
                    .idempotencyKey(idempotencyKey)
                    .amount(request.getAmount())
                    .status(result.getStatus())
                    .build();
                transactionRepository.save(transaction);
                
                return result;
                
            } finally {
                distributedLock.unlock(lockKey);
            }
        } else {
            // Another request is processing, wait and retry
            try {
                Thread.sleep(100);
                return processPayment(request);  // Retry
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new PaymentProcessingException("Failed to acquire lock");
            }
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Idempotency Technology-Level

**Normal Flow: First Request**

```
Step 1: Client Sends Payment Request
┌─────────────────────────────────────────────────────────────┐
│ POST /v1/payments                                           │
│ Idempotency-Key: idemp_abc123                               │
│ Body: { amount: 10000, payment_method_id: "pm_xyz" }       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Redis Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET idempotency:idemp_abc123                         │
│ Result: MISS (new request)                                  │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Check Database
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT * FROM transactions                      │
│              WHERE idempotency_key = 'idemp_abc123'         │
│ Result: 0 rows (not processed before)                       │
│ Latency: ~5ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Acquire Distributed Lock
┌─────────────────────────────────────────────────────────────┐
│ Redis: SET lock:idempotency:idemp_abc123 <uuid> EX 5        │
│ Result: OK (lock acquired)                                  │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Process Payment
┌─────────────────────────────────────────────────────────────┐
│ Payment Processor API: Charge $100                         │
│ Result: Success, transaction_id: "pay_xyz789"              │
│ Latency: ~500ms (payment processor)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Store Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                              │
│ PostgreSQL: INSERT INTO transactions                       │
│              (transaction_id, idempotency_key, amount, ...) │
│ PostgreSQL: COMMIT TRANSACTION                              │
│ Latency: ~10ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 7: Cache Result
┌─────────────────────────────────────────────────────────────┐
│ Redis: SET idempotency:idemp_abc123 <result> EX 86400      │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 8: Release Lock and Return
┌─────────────────────────────────────────────────────────────┐
│ Redis: DEL lock:idempotency:idemp_abc123                    │
│ Response: 200 OK, PaymentResult                            │
│ Total latency: ~520ms                                       │
└─────────────────────────────────────────────────────────────┘
```

**Retry Flow: Duplicate Request**

```
Step 1: Client Retries (Network Timeout)
┌─────────────────────────────────────────────────────────────┐
│ POST /v1/payments                                           │
│ Idempotency-Key: idemp_abc123 (same key)                    │
│ Body: { amount: 10000, payment_method_id: "pm_xyz" }       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Redis Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET idempotency:idemp_abc123                         │
│ Result: HIT (cached from first request)                    │
│ Value: {                                                     │
│   "payment_id": "pay_xyz789",                              │
│   "status": "succeeded",                                    │
│   "amount": 10000                                           │
│ }                                                            │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Cached Result
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK, PaymentResult (cached)                   │
│ Total latency: ~2ms (vs ~520ms for new request)            │
│ No duplicate charge performed                              │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Redis Down**

```
Step 1: Redis Cache Unavailable
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET idempotency:idemp_abc123                         │
│ Result: Connection timeout                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Fallback to Database
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT * FROM transactions                      │
│              WHERE idempotency_key = 'idemp_abc123'         │
│ Result: 1 row (found)                                       │
│ Decision: Return existing transaction (no duplicate charge)│
│ Latency: ~5ms (vs ~1ms with Redis)                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Double-Entry Bookkeeping

### A) CONCEPT: What is Double-Entry Bookkeeping?

Double-entry bookkeeping is an accounting method where every transaction affects at least two accounts, with equal debits and credits. This ensures:

1. **Accuracy**: Errors are easier to detect
2. **Completeness**: All transactions are recorded
3. **Compliance**: Required by financial regulations
4. **Audit Trail**: Complete transaction history

**Accounting Equation:**
```
Assets = Liabilities + Equity + Revenue - Expenses
```

**Rules:**
- Every transaction has equal debits and credits
- Total debits = Total credits
- Accounts must always balance

### B) OUR USAGE: How We Implement Double-Entry

**Account Types:**
- **Assets**: Customer accounts, cash
- **Liabilities**: Accounts payable, refunds payable
- **Equity**: Company equity
- **Revenue**: Payment revenue
- **Expenses**: Processing fees, chargebacks

**Transaction Flow:**

```java
@Service
public class LedgerService {
    
    private final LedgerRepository ledgerRepository;
    private final AccountRepository accountRepository;
    
    @Transactional
    public void recordPayment(Transaction transaction) {
        // 1. Debit: Customer account (asset increases)
        LedgerEntry debitEntry = LedgerEntry.builder()
            .transactionId(transaction.getTransactionId())
            .accountId("customer_account_" + transaction.getCustomerId())
            .accountType(AccountType.ASSET)
            .debit(transaction.getAmount())
            .credit(null)
            .description("Payment received from customer")
            .build();
        
        // Update customer account balance
        Account customerAccount = accountRepository.findById(
            "customer_account_" + transaction.getCustomerId()
        );
        customerAccount.setBalance(
            customerAccount.getBalance().add(transaction.getAmount())
        );
        accountRepository.save(customerAccount);
        
        debitEntry.setBalance(customerAccount.getBalance());
        ledgerRepository.save(debitEntry);
        
        // 2. Credit: Revenue account (revenue increases)
        LedgerEntry creditEntry = LedgerEntry.builder()
            .transactionId(transaction.getTransactionId())
            .accountId("revenue_account")
            .accountType(AccountType.REVENUE)
            .debit(null)
            .credit(transaction.getAmount())
            .description("Payment revenue")
            .build();
        
        // Update revenue account balance
        Account revenueAccount = accountRepository.findById("revenue_account");
        revenueAccount.setBalance(
            revenueAccount.getBalance().add(transaction.getAmount())
        );
        accountRepository.save(revenueAccount);
        
        creditEntry.setBalance(revenueAccount.getBalance());
        ledgerRepository.save(creditEntry);
        
        // 3. Verify balance
        verifyBalance(transaction.getTransactionId());
    }
    
    private void verifyBalance(String transactionId) {
        List<LedgerEntry> entries = ledgerRepository.findByTransactionId(transactionId);
        
        BigDecimal totalDebits = entries.stream()
            .filter(e -> e.getDebit() != null)
            .map(LedgerEntry::getDebit)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        BigDecimal totalCredits = entries.stream()
            .filter(e -> e.getCredit() != null)
            .map(LedgerEntry::getCredit)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        if (!totalDebits.equals(totalCredits)) {
            throw new LedgerImbalanceException(
                String.format("Transaction %s: Debits (%s) != Credits (%s)",
                    transactionId, totalDebits, totalCredits)
            );
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Ledger Technology-Level

**Normal Flow: Payment Recording**

```
Step 1: Payment Succeeds
┌─────────────────────────────────────────────────────────────┐
│ Payment: $100 from customer_123                             │
│ Transaction ID: txn_abc123                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Begin Database Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                              │
│ Isolation: SERIALIZABLE (strongest isolation)              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Create Debit Entry
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: INSERT INTO ledger_entries                      │
│              (transaction_id, account_id, debit, ...)        │
│              VALUES ('txn_abc123',                          │
│                      'customer_account_123',                │
│                      100.00, NULL, ...)                     │
│                                                              │
│ Update customer account balance:                           │
│   Old balance: $500.00                                      │
│   New balance: $600.00                                      │
│ Latency: ~5ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Create Credit Entry
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: INSERT INTO ledger_entries                      │
│              (transaction_id, account_id, credit, ...)      │
│              VALUES ('txn_abc123',                          │
│                      'revenue_account',                     │
│                      NULL, 100.00, ...)                     │
│                                                              │
│ Update revenue account balance:                            │
│   Old balance: $10,000.00                                    │
│   New balance: $10,100.00                                   │
│ Latency: ~5ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Verify Balance
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT SUM(debit), SUM(credit)                 │
│              FROM ledger_entries                            │
│              WHERE transaction_id = 'txn_abc123'            │
│ Result: Debits = $100.00, Credits = $100.00                │
│ Balance: ✓ Balanced                                         │
│ Latency: ~2ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Commit Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: COMMIT TRANSACTION                             │
│ Result: Success                                             │
│ Total latency: ~15ms                                         │
│ Both entries persisted atomically                           │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Imbalance Detected**

```
Step 1: Imbalance Detected
┌─────────────────────────────────────────────────────────────┐
│ Verification: Debits = $100.00, Credits = $99.00           │
│ Error: Imbalance detected                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Rollback Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: ROLLBACK TRANSACTION                           │
│ Result: All changes reverted                                │
│                                                              │
│ Alert: LedgerImbalanceException                            │
│ Action: Log error, alert operations team                   │
│ Impact: Transaction not recorded, payment may be retried   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Fraud Detection

### A) CONCEPT: What is Fraud Detection?

Fraud detection identifies suspicious transactions before they're processed, preventing chargebacks and losses. It uses:

1. **Rule-Based Detection**: Predefined rules (velocity, amount limits)
2. **Machine Learning**: ML models trained on historical fraud data
3. **Behavioral Analysis**: User behavior patterns
4. **Device Fingerprinting**: Device and location analysis

### B) OUR USAGE: Real-Time Fraud Scoring

**Fraud Check Flow:**

```java
@Service
public class FraudDetectionService {
    
    private final MLModelService mlModelService;
    private final RuleEngine ruleEngine;
    
    public FraudCheckResult checkTransaction(Transaction transaction) {
        // 1. Rule-based checks (fast, deterministic)
        RuleCheckResult ruleResult = ruleEngine.check(transaction);
        if (ruleResult.isBlocked()) {
            return FraudCheckResult.blocked(ruleResult.getReason());
        }
        
        // 2. ML model scoring (slower, probabilistic)
        MLScore mlScore = mlModelService.score(transaction);
        
        // 3. Combine results
        double finalScore = combineScores(ruleResult, mlScore);
        
        if (finalScore > 0.8) {
            return FraudCheckResult.blocked("High fraud risk");
        } else if (finalScore > 0.5) {
            return FraudCheckResult.review("Medium fraud risk");
        } else {
            return FraudCheckResult.allowed("Low fraud risk");
        }
    }
    
    private double combineScores(RuleCheckResult rule, MLScore ml) {
        // Weighted combination
        double ruleWeight = 0.4;
        double mlWeight = 0.6;
        
        return (rule.getScore() * ruleWeight) + (ml.getScore() * mlWeight);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Fraud Check Technology-Level

**Normal Flow: Low Risk Transaction**

```
Step 1: Payment Request
┌─────────────────────────────────────────────────────────────┐
│ Transaction: $50 from customer_123                         │
│ Payment method: pm_abc123 (saved card)                     │
│ Location: San Francisco, CA (usual location)              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Rule-Based Checks
┌─────────────────────────────────────────────────────────────┐
│ Check 1: Velocity (transactions per hour)                  │
│   Result: 2 transactions/hour (normal)                     │
│                                                              │
│ Check 2: Amount limit                                       │
│   Result: $50 < $1,000 limit (normal)                      │
│                                                              │
│ Check 3: Location                                           │
│   Result: Same city as usual (normal)                      │
│                                                              │
│ Rule Score: 0.1 (low risk)                                  │
│ Latency: ~5ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: ML Model Scoring
┌─────────────────────────────────────────────────────────────┐
│ ML Model: Load features                                     │
│   - Transaction amount: $50                                  │
│   - Time of day: 14:30                                      │
│   - Device fingerprint: device_xyz                          │
│   - IP address: 192.168.1.1                                 │
│   - Historical behavior: 100 transactions, 0 chargebacks    │
│                                                              │
│ Model Inference:                                             │
│   Score: 0.15 (low fraud probability)                       │
│ Latency: ~50ms (ML inference)                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Combine Scores
┌─────────────────────────────────────────────────────────────┐
│ Final Score = (0.1 × 0.4) + (0.15 × 0.6) = 0.13            │
│ Decision: ALLOW (score < 0.5 threshold)                    │
│ Total latency: ~55ms                                         │
└─────────────────────────────────────────────────────────────┘
```

**High Risk Flow: Suspicious Transaction**

```
Step 1: Suspicious Payment Request
┌─────────────────────────────────────────────────────────────┐
│ Transaction: $5,000 from customer_123                       │
│ Payment method: pm_new456 (new card, added today)          │
│ Location: Moscow, Russia (unusual location)                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Rule-Based Checks
┌─────────────────────────────────────────────────────────────┐
│ Check 1: Velocity                                           │
│   Result: 10 transactions/hour (high!)                      │
│                                                              │
│ Check 2: Amount limit                                       │
│   Result: $5,000 > $1,000 limit (flagged)                  │
│                                                              │
│ Check 3: Location                                           │
│   Result: Different country (unusual)                       │
│                                                              │
│ Check 4: New payment method                                 │
│   Result: Card added < 24 hours ago (risky)                 │
│                                                              │
│ Rule Score: 0.85 (high risk)                                 │
│ Decision: BLOCK (multiple red flags)                         │
│ Latency: ~5ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Blocked Result
┌─────────────────────────────────────────────────────────────┐
│ Response: Transaction blocked                              │
│ Reason: "Multiple fraud indicators detected"               │
│ Action: Transaction not processed, customer notified      │
│ Total latency: ~5ms (early blocking saves ML inference)    │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Payment Processing Flow

### A) CONCEPT: Payment Processor Integration

Payment processors (Stripe, PayPal, Square) handle the actual charging of payment methods. Our system:

1. **Abstracts Differences**: Unified API across processors
2. **Handles Failures**: Retries, failover to backup processor
3. **Tracks State**: Payment status, webhooks, reconciliation

### B) OUR USAGE: Payment Processor Gateway

```java
@Service
public class PaymentProcessorGateway {
    
    private final StripeClient stripeClient;
    private final PayPalClient paypalClient;
    private final CircuitBreaker circuitBreaker;
    
    public PaymentResult charge(PaymentRequest request) {
        // Try primary processor (Stripe)
        try {
            return circuitBreaker.executeSupplier(() -> {
                return stripeClient.charge(request);
            });
        } catch (CircuitBreakerOpenException e) {
            // Failover to backup processor
            log.warn("Stripe circuit breaker open, failing over to PayPal");
            return paypalClient.charge(request);
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Payment Processing

**Normal Flow: Successful Charge**

```
Step 1: Charge Request
┌─────────────────────────────────────────────────────────────┐
│ Payment: $100, Card: pm_abc123                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Call Payment Processor
┌─────────────────────────────────────────────────────────────┐
│ Stripe API: POST /v1/charges                                │
│ Body: { amount: 10000, payment_method: "pm_abc123" }       │
│ Latency: ~500ms (network + processor)                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Processor Response
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                            │
│ {                                                           │
│   "id": "ch_stripe_xyz789",                                │
│   "status": "succeeded",                                    │
│   "amount": 10000                                           │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Store Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: INSERT INTO transactions                       │
│              (processor_transaction_id, status, ...)        │
│              VALUES ('ch_stripe_xyz789', 'succeeded', ...) │
│ Latency: ~10ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Update Ledger
┌─────────────────────────────────────────────────────────────┐
│ Record double-entry entries                                 │
│ Latency: ~15ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Return Success
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                            │
│ Total latency: ~540ms                                        │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Card Declined**

```
Step 1: Charge Request
┌─────────────────────────────────────────────────────────────┐
│ Payment: $100, Card: pm_abc123                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Processor Declines
┌─────────────────────────────────────────────────────────────┐
│ Stripe API: POST /v1/charges                                │
│ Response: 402 Payment Required                              │
│ {                                                           │
│   "error": {                                                │
│     "code": "card_declined",                                │
│     "decline_code": "insufficient_funds"                   │
│   }                                                         │
│ }                                                            │
│ Latency: ~500ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Store Failed Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: INSERT INTO transactions                       │
│              (status, failure_code, ...)                    │
│              VALUES ('failed', 'insufficient_funds', ...)   │
│ Latency: ~10ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Return Error
┌─────────────────────────────────────────────────────────────┐
│ Response: 400 Bad Request                                   │
│ {                                                           │
│   "error": {                                                │
│     "code": "CARD_DECLINED",                                │
│     "message": "Your card was declined."                   │
│   }                                                         │
│ }                                                            │
│ Total latency: ~520ms                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Idempotency | Redis + PostgreSQL | 24-hour TTL, distributed locking |
| Ledger | PostgreSQL (ACID) | Double-entry, balance verification |
| Fraud Detection | ML Models + Rules | Real-time scoring, < 500ms latency |
| Payment Processing | Processor Gateway | Circuit breakers, failover |
