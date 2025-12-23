# 🏧 ATM System - Code Walkthrough

## Building From Scratch: Step-by-Step

### Phase 1: Define the Domain (Enums First)

```java
// Step 1: TransactionType enum
// File: com/atm/enums/TransactionType.java

package com.atm.enums;

public enum TransactionType {
    BALANCE_INQUIRY,
    WITHDRAWAL,
    DEPOSIT,
    TRANSFER,
    PIN_CHANGE
}
```

**Why these transaction types?**
- Core ATM operations
- Each has different workflow
- Easy to add new types later

```java
// Step 2: ATMState enum
// File: com/atm/enums/ATMState.java

public enum ATMState {
    IDLE,                     // Waiting for card
    CARD_INSERTED,            // Card in, waiting for PIN
    AUTHENTICATED,            // PIN verified
    TRANSACTION_IN_PROGRESS,  // Processing transaction
    OUT_OF_SERVICE,           // Maintenance
    OUT_OF_CASH               // No cash available
}
```

**State machine reasoning:**
- ATM can only be in one state at a time
- States determine valid operations
- Clear transitions between states

---

### Phase 2: Build Account System

```java
// Step 3: Account abstract class
// File: com/atm/models/Account.java

package com.atm.models;

import java.util.concurrent.locks.ReentrantLock;

public abstract class Account {
    
    protected final String accountNumber;
    protected final AccountType type;
    protected final Customer owner;
    protected double balance;
    
    protected final ReentrantLock lock;
```

**Line-by-line explanation:**

- `protected final String accountNumber` - Unique identifier, immutable
- `protected double balance` - Mutable, changes with transactions
- `protected final ReentrantLock lock` - Thread safety for concurrent access

```java
    public boolean withdraw(double amount) {
        lock.lock();
        try {
            if (amount <= 0) {
                System.out.println("Invalid withdrawal amount");
                return false;
            }
            
            if (!canWithdraw(amount)) {
                System.out.println("Insufficient funds or limit exceeded");
                return false;
            }
            
            balance -= amount;
            return true;
        } finally {
            lock.unlock();
        }
    }
```

**Why lock in try-finally?**
- Guarantees lock release even if exception occurs
- Prevents deadlock from forgotten unlock
- Standard pattern for lock usage

**Why `canWithdraw` is separate?**
- Subclasses override for specific rules
- CheckingAccount: daily limit
- SavingsAccount: minimum balance, monthly limit

```java
// Step 4: CheckingAccount
// File: com/atm/models/CheckingAccount.java

public class CheckingAccount extends Account {
    
    private double dailyWithdrawalLimit;
    private double withdrawnToday;
    
    @Override
    protected boolean canWithdraw(double amount) {
        // Check 1: Sufficient balance
        if (balance < amount) {
            return false;
        }
        
        // Check 2: Daily limit
        if (withdrawnToday + amount > dailyWithdrawalLimit) {
            System.out.println("Daily withdrawal limit exceeded");
            return false;
        }
        
        return true;
    }
    
    @Override
    public boolean withdraw(double amount) {
        boolean success = super.withdraw(amount);
        if (success) {
            withdrawnToday += amount;  // Track daily usage
        }
        return success;
    }
}
```

**Daily limit tracking:**
- `withdrawnToday` accumulates during the day
- Reset at midnight (by scheduled job)
- Prevents excessive withdrawals

---

### Phase 3: Build Card and Authentication

```java
// Step 5: Card class
// File: com/atm/models/Card.java

package com.atm.models;

import java.time.LocalDate;
import java.util.*;

public class Card {
    
    private final String cardNumber;
    private final String customerName;
    private final LocalDate expirationDate;
    private String pin;
    private boolean isBlocked;
    private int failedPinAttempts;
    
    private final List<Account> linkedAccounts;
    
    private static final int MAX_PIN_ATTEMPTS = 3;
```

**Security considerations:**
- `pin` should be hashed in real system
- `isBlocked` prevents further use
- `failedPinAttempts` tracks security violations

```java
    public boolean validatePin(String enteredPin) {
        // Check if card is blocked
        if (isBlocked) {
            System.out.println("Card is blocked");
            return false;
        }
        
        // Validate PIN
        if (pin.equals(enteredPin)) {
            failedPinAttempts = 0;  // Reset on success
            return true;
        }
        
        // Wrong PIN
        failedPinAttempts++;
        
        // Block after max attempts
        if (failedPinAttempts >= MAX_PIN_ATTEMPTS) {
            isBlocked = true;
            System.out.println("Card blocked after " + MAX_PIN_ATTEMPTS + " failed attempts");
        }
        
        return false;
    }
```

**PIN validation flow:**

```
validatePin("1234")
         │
         ▼
    ┌────────────┐     Yes
    │ isBlocked? │─────────► Return false
    └─────┬──────┘
          │ No
          ▼
    ┌────────────┐     Yes
    │ PIN match? │─────────► Reset attempts, Return true
    └─────┬──────┘
          │ No
          ▼
    failedAttempts++
          │
          ▼
    ┌────────────────┐     Yes
    │ attempts >= 3? │─────────► Block card
    └────────────────┘
          │
          ▼
      Return false
```

---

### Phase 4: Build Hardware Components

```java
// Step 6: CashDispenser
// File: com/atm/hardware/CashDispenser.java

package com.atm.hardware;

import java.util.*;

public class CashDispenser {
    
    // Denomination -> Count
    private final Map<Integer, Integer> cashInventory;
    private static final int[] DENOMINATIONS = {100, 50, 20, 10, 5, 1};
```

**Why LinkedHashMap?**
- Maintains insertion order
- Important for greedy algorithm (largest first)
- Predictable iteration

```java
    public synchronized boolean dispenseCash(double amount) {
        int amountInt = (int) amount;
        
        // Calculate which bills to use
        Map<Integer, Integer> billsToDispense = calculateBills(amountInt);
        if (billsToDispense == null) {
            System.out.println("Cannot dispense $" + amount);
            return false;
        }
        
        // Update inventory
        for (Map.Entry<Integer, Integer> entry : billsToDispense.entrySet()) {
            int denomination = entry.getKey();
            int count = entry.getValue();
            cashInventory.merge(denomination, -count, Integer::sum);
        }
        
        // Display what's being dispensed
        System.out.println("Dispensing:");
        for (Map.Entry<Integer, Integer> entry : billsToDispense.entrySet()) {
            if (entry.getValue() > 0) {
                System.out.println("  " + entry.getValue() + " × $" + entry.getKey());
            }
        }
        
        return true;
    }
```

**Why synchronized?**
- Multiple ATM transactions could access same dispenser
- Prevents race condition on inventory
- Ensures atomic update

```java
    private Map<Integer, Integer> calculateBills(int amount) {
        Map<Integer, Integer> result = new LinkedHashMap<>();
        int remaining = amount;
        
        // Greedy algorithm: use largest bills first
        for (int denomination : DENOMINATIONS) {
            int available = cashInventory.getOrDefault(denomination, 0);
            int needed = remaining / denomination;
            int toUse = Math.min(needed, available);
            
            if (toUse > 0) {
                result.put(denomination, toUse);
                remaining -= toUse * denomination;
            }
        }
        
        // Check if exact amount can be dispensed
        if (remaining > 0) {
            return null;  // Cannot dispense
        }
        
        return result;
    }
```

**Greedy algorithm example:**

```
Amount: $270
Inventory: $100×10, $50×10, $20×10, $10×10

Step 1: $100 bills
  needed = 270 / 100 = 2
  toUse = min(2, 10) = 2
  remaining = 270 - 200 = 70

Step 2: $50 bills
  needed = 70 / 50 = 1
  toUse = min(1, 10) = 1
  remaining = 70 - 50 = 20

Step 3: $20 bills
  needed = 20 / 20 = 1
  toUse = min(1, 10) = 1
  remaining = 20 - 20 = 0

Result: 2×$100 + 1×$50 + 1×$20 = $270 ✓
```

---

### Phase 5: Build Transactions

```java
// Step 7: Transaction abstract class
// File: com/atm/models/Transaction.java

public abstract class Transaction {
    
    protected final String transactionId;
    protected final TransactionType type;
    protected final Account account;
    protected final double amount;
    protected final LocalDateTime timestamp;
    
    protected TransactionStatus status;
    protected String failureReason;
    
    public abstract boolean execute();
```

**Why abstract?**
- Common fields for all transactions
- Each type has different execution logic
- Polymorphism for transaction processing

```java
// Step 8: WithdrawalTransaction
// File: com/atm/models/WithdrawalTransaction.java

public class WithdrawalTransaction extends Transaction {
    
    private final CashDispenser cashDispenser;
    
    @Override
    public boolean execute() {
        // Step 1: Validate amount
        if (amount <= 0 || amount % 20 != 0) {
            fail("Amount must be positive and in multiples of $20");
            return false;
        }
        
        // Step 2: Check if ATM has enough cash
        if (!cashDispenser.canDispense(amount)) {
            fail("ATM cannot dispense this amount");
            return false;
        }
        
        // Step 3: Withdraw from account
        if (!account.withdraw(amount)) {
            fail("Insufficient funds or limit exceeded");
            return false;
        }
        
        // Step 4: Dispense cash
        if (!cashDispenser.dispenseCash(amount)) {
            // Rollback account withdrawal
            account.deposit(amount);
            fail("Failed to dispense cash");
            return false;
        }
        
        // Step 5: Record transaction
        account.addTransaction(this);
        complete();
        
        return true;
    }
}
```

**Withdrawal sequence with rollback:**

```
execute()
    │
    ├── Validate amount
    │       │ Invalid
    │       └──────► fail(), return false
    │
    ├── Check ATM cash
    │       │ Not enough
    │       └──────► fail(), return false
    │
    ├── Withdraw from account
    │       │ Failed
    │       └──────► fail(), return false
    │
    ├── Dispense cash
    │       │ Failed
    │       └──────► ROLLBACK: deposit back
    │                fail(), return false
    │
    └── Success
            │
            └──────► complete(), return true
```

---

### Phase 6: Build Controller

```java
// Step 9: ATMController
// File: com/atm/controller/ATMController.java

public class ATMController {
    
    private final ATM atm;
    private ATMState state;
    private Card currentCard;
    private Account selectedAccount;
```

**State management:**
- `state` tracks ATM lifecycle
- `currentCard` holds authenticated card
- `selectedAccount` holds selected account for transactions

```java
    public boolean insertCard(Card card) {
        // Validate current state
        if (state != ATMState.IDLE) {
            atm.getScreen().displayError("ATM is busy");
            return false;
        }
        
        // Read card
        Card readCard = atm.getCardReader().readCard(card);
        if (readCard == null) {
            return false;
        }
        
        // Update state
        currentCard = readCard;
        state = ATMState.CARD_INSERTED;
        atm.getScreen().displayMessage("Card accepted.\nPlease enter your PIN.");
        
        return true;
    }
```

**State transition on card insert:**

```
State: IDLE
    │
    │ insertCard(card)
    │
    ├── state != IDLE?
    │       │ Yes
    │       └──► Display error, return false
    │
    ├── cardReader.readCard(card)
    │       │ null (invalid/expired/blocked)
    │       └──► return false
    │
    └── Success
            │
            ├── currentCard = card
            ├── state = CARD_INSERTED
            └── Display "Enter PIN"
```

```java
    public boolean authenticate(String pin) {
        if (state != ATMState.CARD_INSERTED) {
            return false;
        }
        
        if (currentCard.validatePin(pin)) {
            state = ATMState.AUTHENTICATED;
            atm.getScreen().displayMessage("Welcome, " + currentCard.getCustomerName());
            return true;
        } else {
            int remaining = currentCard.getRemainingAttempts();
            if (remaining > 0) {
                atm.getScreen().displayError(
                    "Incorrect PIN. " + remaining + " attempts remaining.");
            } else {
                atm.getScreen().displayError(
                    "Card blocked. Please contact your bank.");
                atm.getCardReader().retainCard();
                state = ATMState.IDLE;
            }
            return false;
        }
    }
```

**Authentication flow:**

```
authenticate("1234")
    │
    ├── state != CARD_INSERTED?
    │       │ Yes
    │       └──► return false
    │
    ├── card.validatePin("1234")
    │       │
    │       ├── Success
    │       │       │
    │       │       ├── state = AUTHENTICATED
    │       │       └── Display "Welcome"
    │       │
    │       └── Failure
    │               │
    │               ├── remaining > 0?
    │               │       │ Yes
    │               │       └── Display "X attempts remaining"
    │               │
    │               └── remaining == 0?
    │                       │ Yes
    │                       ├── Retain card
    │                       └── state = IDLE
    │
    └── return result
```

---

## Testing Approach

### Unit Tests

```java
// CardTest.java
public class CardTest {
    
    @Test
    void testValidPinAuthentication() {
        Card card = new Card("1234567890123456", "Alice", "1234", 
                            LocalDate.now().plusYears(5));
        
        assertTrue(card.validatePin("1234"));
        assertFalse(card.isBlocked());
    }
    
    @Test
    void testInvalidPinAuthentication() {
        Card card = new Card("1234567890123456", "Alice", "1234", 
                            LocalDate.now().plusYears(5));
        
        assertFalse(card.validatePin("0000"));
        assertEquals(2, card.getRemainingAttempts());
    }
    
    @Test
    void testCardBlockedAfterThreeFailedAttempts() {
        Card card = new Card("1234567890123456", "Alice", "1234", 
                            LocalDate.now().plusYears(5));
        
        card.validatePin("0000");  // Attempt 1
        card.validatePin("0000");  // Attempt 2
        card.validatePin("0000");  // Attempt 3
        
        assertTrue(card.isBlocked());
        assertFalse(card.validatePin("1234"));  // Even correct PIN fails
    }
    
    @Test
    void testExpiredCard() {
        Card card = new Card("1234567890123456", "Alice", "1234", 
                            LocalDate.now().minusDays(1));  // Expired yesterday
        
        assertTrue(card.isExpired());
    }
}
```

```java
// AccountTest.java
public class AccountTest {
    
    @Test
    void testWithdrawalSuccess() {
        Customer customer = new Customer("C001", "Alice", "alice@test.com", 
                                        "555-1234", "123 Main St");
        CheckingAccount account = new CheckingAccount("ACC001", customer, 1000.00);
        
        assertTrue(account.withdraw(200.00));
        assertEquals(800.00, account.getBalance(), 0.01);
    }
    
    @Test
    void testWithdrawalInsufficientFunds() {
        Customer customer = new Customer("C001", "Alice", "alice@test.com", 
                                        "555-1234", "123 Main St");
        CheckingAccount account = new CheckingAccount("ACC001", customer, 100.00);
        
        assertFalse(account.withdraw(200.00));
        assertEquals(100.00, account.getBalance(), 0.01);
    }
    
    @Test
    void testDailyWithdrawalLimit() {
        Customer customer = new Customer("C001", "Alice", "alice@test.com", 
                                        "555-1234", "123 Main St");
        CheckingAccount account = new CheckingAccount("ACC001", customer, 5000.00);
        account.setDailyWithdrawalLimit(500.00);
        
        assertTrue(account.withdraw(300.00));
        assertTrue(account.withdraw(200.00));
        assertFalse(account.withdraw(100.00));  // Exceeds daily limit
    }
}
```

```java
// CashDispenserTest.java
public class CashDispenserTest {
    
    @Test
    void testDispenseExactAmount() {
        CashDispenser dispenser = new CashDispenser();
        
        assertTrue(dispenser.canDispense(270.00));
        assertTrue(dispenser.dispenseCash(270.00));
    }
    
    @Test
    void testCannotDispenseOddAmount() {
        CashDispenser dispenser = new CashDispenser();
        
        // If smallest denomination is $1, this should work
        // If smallest is $5, amounts not divisible by 5 fail
        assertTrue(dispenser.canDispense(17.00));  // With $1 bills
    }
    
    @Test
    void testInventoryDecreasesAfterDispense() {
        CashDispenser dispenser = new CashDispenser();
        double initialCash = dispenser.getTotalCash();
        
        dispenser.dispenseCash(100.00);
        
        assertEquals(initialCash - 100.00, dispenser.getTotalCash(), 0.01);
    }
}
```

### Integration Tests

```java
// ATMIntegrationTest.java
public class ATMIntegrationTest {
    
    private Bank bank;
    private ATM atm;
    private Card card;
    private CheckingAccount account;
    
    @BeforeEach
    void setUp() {
        Bank.resetInstance();
        bank = Bank.getInstance("Test Bank");
        
        Customer customer = bank.createCustomer("Alice", "alice@test.com", 
                                               "555-1234", "123 Main St");
        account = bank.createCheckingAccount(customer, 1000.00);
        card = bank.issueCard(customer, "1234");
        
        atm = bank.createATM("Test Location");
    }
    
    @Test
    void testCompleteWithdrawalFlow() {
        ATMController controller = atm.getController();
        
        // Insert card
        assertTrue(controller.insertCard(card));
        assertEquals(ATMState.CARD_INSERTED, controller.getState());
        
        // Authenticate
        assertTrue(controller.authenticate("1234"));
        assertEquals(ATMState.AUTHENTICATED, controller.getState());
        
        // Select account
        assertTrue(controller.selectAccount(0));
        
        // Withdraw
        assertTrue(controller.withdraw(200.00));
        assertEquals(800.00, account.getBalance(), 0.01);
        
        // End session
        controller.endSession();
        assertEquals(ATMState.IDLE, controller.getState());
    }
    
    @Test
    void testFailedAuthenticationFlow() {
        ATMController controller = atm.getController();
        
        controller.insertCard(card);
        
        // Wrong PIN 3 times
        assertFalse(controller.authenticate("0000"));
        assertFalse(controller.authenticate("0000"));
        assertFalse(controller.authenticate("0000"));
        
        // Card should be blocked and retained
        assertTrue(card.isBlocked());
        assertEquals(ATMState.IDLE, controller.getState());
    }
}
```

---

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `validatePin()` | O(1) | String comparison |
| `withdraw()` | O(1) | Balance check and update |
| `deposit()` | O(1) | Balance update |
| `calculateBills()` | O(D) | D = number of denominations |
| `dispenseCash()` | O(D) | Update inventory |
| `findAccount()` | O(1) | HashMap lookup |

### Space Complexity

| Data Structure | Space | Purpose |
|----------------|-------|---------|
| `cashInventory` | O(D) | D = denominations |
| `linkedAccounts` | O(A) | A = accounts per card |
| `transactions` | O(T) | T = transaction history |

### Bottlenecks at Scale

**High transaction volume:**
- Account lock contention
- Solution: Partition accounts, use optimistic locking

**Multiple ATMs:**
- Cash inventory synchronization
- Solution: Each ATM has local inventory, periodic sync

---

## Interview Follow-ups with Answers

### Q1: How would you handle network failures?

```java
public class ResilientTransaction {
    private static final int MAX_RETRIES = 3;
    
    public boolean executeWithRetry(Transaction transaction) {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                return transaction.execute();
            } catch (NetworkException e) {
                if (attempt == MAX_RETRIES) {
                    // Store for later processing
                    pendingTransactions.add(transaction);
                    return false;
                }
                Thread.sleep(1000 * attempt);  // Exponential backoff
            }
        }
        return false;
    }
}
```

### Q2: How would you implement transaction limits?

```java
public class TransactionLimitService {
    private final Map<String, DailyLimits> accountLimits;
    
    public boolean checkLimit(Account account, TransactionType type, double amount) {
        DailyLimits limits = accountLimits.get(account.getAccountNumber());
        
        switch (type) {
            case WITHDRAWAL:
                return limits.getRemainingWithdrawal() >= amount;
            case TRANSFER:
                return limits.getRemainingTransfer() >= amount;
            default:
                return true;
        }
    }
    
    public void recordTransaction(Account account, TransactionType type, double amount) {
        DailyLimits limits = accountLimits.get(account.getAccountNumber());
        limits.record(type, amount);
    }
}
```

### Q3: How would you add fraud detection?

```java
public class FraudDetectionService {
    
    public boolean isSuspicious(Transaction transaction) {
        // Check 1: Unusual amount
        if (transaction.getAmount() > getAverageAmount(transaction.getAccount()) * 5) {
            return true;
        }
        
        // Check 2: Unusual location
        if (!isNearUsualLocations(transaction)) {
            return true;
        }
        
        // Check 3: Rapid transactions
        if (getRecentTransactionCount(transaction.getAccount()) > 5) {
            return true;
        }
        
        return false;
    }
    
    public void handleSuspiciousTransaction(Transaction transaction) {
        // Send alert
        notificationService.sendFraudAlert(transaction.getAccount().getOwner());
        
        // Require additional verification
        requireOTP(transaction);
    }
}
```

### Q4: How would you handle ATM maintenance?

```java
public class MaintenanceService {
    
    public void scheduleMaintenanceMode(ATM atm, LocalDateTime startTime, 
                                        Duration duration) {
        // Complete current transaction
        atm.getController().waitForCurrentTransaction();
        
        // Set out of service
        atm.getController().setState(ATMState.OUT_OF_SERVICE);
        
        // Schedule return to service
        scheduler.schedule(() -> {
            atm.getController().setState(ATMState.IDLE);
        }, startTime.plus(duration));
    }
    
    public void refillCash(ATM atm, Map<Integer, Integer> bills) {
        CashDispenser dispenser = atm.getCashDispenser();
        for (Map.Entry<Integer, Integer> entry : bills.entrySet()) {
            dispenser.refill(entry.getKey(), entry.getValue());
        }
    }
}
```

### Q5: What would you do differently with more time?

1. **Add audit logging** - Track all operations for compliance
2. **Add encryption** - Encrypt PINs and sensitive data
3. **Add monitoring** - Real-time ATM health monitoring
4. **Add analytics** - Usage patterns, peak times
5. **Add mobile integration** - Cardless transactions via app
6. **Add receipt printing** - Physical and digital receipts
7. **Add multi-language support** - Internationalization

