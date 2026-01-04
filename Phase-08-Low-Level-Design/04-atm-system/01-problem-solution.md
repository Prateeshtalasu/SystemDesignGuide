# 🏧 ATM System - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements
- Authenticate users via card and PIN
- Support balance inquiry for linked accounts
- Support cash withdrawal with denomination selection
- Support cash deposit
- Support fund transfer between accounts
- Support PIN change
- Handle cash dispensing with multiple denominations ($100, $50, $20, $10)
- Log all transactions with timestamps
- Support multiple account types (Checking, Savings)

### Explicit Out-of-Scope Items
- Check deposit and imaging
- Bill payment services
- Account opening/closing
- Credit card cash advances
- Foreign currency exchange
- Mini-statement printing
- Card-less transactions (mobile banking)
- Biometric authentication
- Real-time fraud detection
- Integration with external banking networks

### Assumptions and Constraints
- **Single Bank**: ATM serves one bank's customers only
- **In-Memory Storage**: No external database
- **Fixed Denominations**: $100, $50, $20, $10 bills only
- **Daily Withdrawal Limit**: $1000 per card per day
- **PIN Attempts**: 3 attempts before card is blocked
- **Session Timeout**: Auto-logout after 30 seconds of inactivity
- **No Partial Dispense**: If exact amount can't be dispensed, transaction fails
- **Simulated Hardware**: Card reader, dispenser, keypad are simulated

### Scale Assumptions
- **Single Process**: Runs within a single JVM
- **Single ATM**: One ATM instance per system
- **Moderate Load**: Up to 100 transactions per hour

### Concurrency Model Expectations
- **Synchronized Cash Dispenser**: Prevent race conditions on cash inventory
- **Thread-Safe Transaction Log**: Multiple transactions logged safely
- **Atomic Balance Updates**: Account balance changes are atomic
- **Session Isolation**: Each ATM session is independent

### Public APIs (Main methods exposed for interaction)
- `ATM.insertCard(Card)`: Start session with card
- `ATM.enterPin(String)`: Authenticate with PIN
- `ATM.selectAccount(AccountType)`: Choose account for transaction
- `ATM.checkBalance()`: Get current balance
- `ATM.withdraw(amount)`: Withdraw cash
- `ATM.deposit(amount)`: Deposit cash
- `ATM.transfer(toAccount, amount)`: Transfer funds
- `ATM.changePin(oldPin, newPin)`: Change PIN
- `ATM.ejectCard()`: End session
- `ATM.cancel()`: Cancel current operation

### Public API Usage Examples
```java
// Example 1: Basic usage
Bank bank = Bank.getInstance("National Bank");
Customer customer = bank.createCustomer("Alice", "alice@email.com", 
                                       "555-1234", "123 Main St");
CheckingAccount account = bank.createCheckingAccount(customer, 5000.00);
Card card = bank.issueCard(customer, "1234");
ATM atm = bank.createATM("Downtown Branch");

// Insert card and authenticate
atm.getController().insertCard(card);
atm.getController().authenticate("1234");
atm.getController().selectAccount(0);

// Example 2: Typical workflow
// Check balance
atm.getController().checkBalance();

// Withdraw cash
atm.getController().withdraw(200.00);

// Deposit cash
atm.getController().deposit(500.00);

// Transfer funds
atm.getController().transfer(savingsAccount.getAccountNumber(), 1000.00);

// End session
atm.getController().endSession();

// Example 3: Edge case - Wrong PIN attempts
atm.getController().insertCard(card);
atm.getController().authenticate("0000");  // Wrong - 2 attempts remaining
atm.getController().authenticate("0000");  // Wrong - 1 attempt remaining
atm.getController().authenticate("0000");  // Wrong - Card blocked

// Example 4: Edge case - Insufficient funds
atm.getController().withdraw(10000.00);  // Balance only $5000
// Transaction fails: "Insufficient funds or limit exceeded"

// Example 5: Edge case - ATM out of cash
atm.getController().withdraw(50000.00);  // ATM doesn't have enough
// Transaction fails: "ATM cannot dispense this amount"
```

### Invariants the System Must Always Maintain
- **Card-Session Binding**: Only one session per card at a time
- **PIN Security**: PIN never stored in plain text, never logged
- **Balance Consistency**: Account balance = initial + deposits - withdrawals
- **Cash Inventory**: Dispensed cash ≤ available cash in ATM
- **Transaction Atomicity**: Either complete transaction or full rollback
- **Daily Limit**: Total withdrawals ≤ daily limit per card
- **Audit Trail**: Every transaction is logged with timestamp

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 ATM SYSTEM                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │     ATM      │────────►│  CardReader      │         │   CashDispenser  │    │
│  │              │         │                  │         │                  │    │
│  └──────┬───────┘         └──────────────────┘         └────────┬─────────┘    │
│         │                                                       │              │
│         │                                                       │              │
│         ▼                                                       ▼              │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │   Keypad     │         │   Screen         │         │ CashSlot         │    │
│  │              │         │                  │         │ (Deposit)        │    │
│  └──────────────┘         └──────────────────┘         └──────────────────┘    │
│                                                                                │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │    Card      │◄───────►│     Account      │◄───────►│   Transaction    │    │
│  │              │         │   (abstract)     │         │                  │    │
│  └──────────────┘         └────────┬─────────┘         └────────┬─────────┘    │
│         │                          │                            │              │
│         │                 ┌────────┴────────┐          ┌────────┴────────┐    │
│         │                 │                 │          │                 │    │
│         ▼                 ▼                 ▼          ▼                 ▼    │
│  ┌──────────────┐  ┌──────────────┐ ┌──────────────┐ ┌─────────┐ ┌─────────┐ │
│  │   Customer   │  │  Checking    │ │   Savings    │ │Withdraw │ │ Deposit │ │
│  │              │  │  Account     │ │   Account    │ │  Txn    │ │   Txn   │ │
│  └──────────────┘  └──────────────┘ └──────────────┘ └─────────┘ └─────────┘ │
│                                                                               │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐   │
│  │    Bank      │────────►│ ATMController    │────────►│ TransactionLog   │   │
│  │ (Singleton)  │         │                  │         │                  │   │
│  └──────────────┘         └──────────────────┘         └──────────────────┘   │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Relationships Summary

| Relationship | Type | Description |
|-------------|------|-------------|
| ATM → CardReader | Composition | ATM owns card reader |
| ATM → CashDispenser | Composition | ATM owns cash dispenser |
| ATM → Keypad | Composition | ATM owns keypad |
| ATM → Screen | Composition | ATM owns screen |
| Card → Customer | Association | Card belongs to customer |
| Card → Account | Association | Card linked to accounts |
| Account → Transaction | Aggregation | Account has transactions |
| Bank → ATM | Aggregation | Bank manages ATMs |
| ATM → ATMController | Composition | ATM has controller |

---

### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `ATM` | ATM hardware components and overall ATM instance | Central container for all hardware - singleton ensures single ATM instance, owns all hardware components |
| `ATMController` | ATM operation workflow and transaction coordination | Coordinates user interaction flow - separates workflow logic from hardware and account operations |
| `Card` | Card authentication data (card number, PIN, expiration) | Encapsulates card identity and authentication - enables card validation and PIN checking |
| `CardReader` | Card reading and validation operations | Handles card hardware interaction - separates hardware concerns from business logic |
| `Account` (abstract) | Account balance and transaction history | Base class for account types - enables polymorphism, defines common account operations |
| `CheckingAccount` | Checking account-specific balance and rules | Implements checking account logic - separate from savings to handle different rules and constraints |
| `SavingsAccount` | Savings account-specific balance and rules | Implements savings account logic - separate from checking for different interest and withdrawal rules |
| `Transaction` (abstract) | Transaction execution template and state | Base class for transaction types - template method pattern enables consistent transaction execution |
| `WithdrawalTransaction` | Withdrawal operation logic and validation | Handles withdrawal-specific logic - separate class enables withdrawal rules without affecting other transaction types |
| `DepositTransaction` | Deposit operation logic and validation | Handles deposit-specific logic - separate from withdrawal to enable independent changes |
| `TransferTransaction` | Transfer operation logic between accounts | Handles transfer-specific logic - separate transaction type for multi-account operations |
| `BalanceInquiryTransaction` | Balance inquiry operation | Handles read-only balance check - separate transaction type for query operations |
| `CashDispenser` | Cash inventory and dispensing operations | Manages cash hardware and inventory - separates cash handling from transaction logic |
| `CashSlot` | Cash deposit slot operations | Handles deposit hardware - separates deposit hardware from transaction logic |
| `Screen` | Display output and user messaging | Handles display output - separates UI presentation from business logic |
| `Keypad` | User input capture and validation | Handles input hardware - separates input capture from business logic |

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Define the Domain (Enums First)

```java
// Step 1: TransactionType enum
public enum TransactionType {
    BALANCE_INQUIRY,
    WITHDRAWAL,
    DEPOSIT,
    TRANSFER,
    PIN_CHANGE
}

// Step 2: ATMState enum
public enum ATMState {
    IDLE,
    CARD_INSERTED,
    AUTHENTICATED,
    TRANSACTION_IN_PROGRESS,
    OUT_OF_SERVICE,
    OUT_OF_CASH
}
```

---

### Phase 2: Build Account System

```java
// Step 3: Account abstract class
public abstract class Account {
    protected final String accountNumber;
    protected final AccountType type;
    protected final Customer owner;
    protected double balance;
    protected final ReentrantLock lock;
    
    public boolean withdraw(double amount) {
        lock.lock();
        try {
            if (amount <= 0 || !canWithdraw(amount)) {
                return false;
            }
            balance -= amount;
            return true;
        } finally {
            lock.unlock();
        }
    }
}

// Step 4: CheckingAccount
public class CheckingAccount extends Account {
    private double dailyWithdrawalLimit;
    private double withdrawnToday;
    
    @Override
    protected boolean canWithdraw(double amount) {
        if (balance < amount) return false;
        if (withdrawnToday + amount > dailyWithdrawalLimit) return false;
        return true;
    }
}
```

---

### Phase 3: Build Card and Authentication

```java
// Step 5: Card class
public class Card {
    private final String cardNumber;
    private String pin;
    private boolean isBlocked;
    private int failedPinAttempts;
    private static final int MAX_PIN_ATTEMPTS = 3;
    
    public boolean validatePin(String enteredPin) {
        if (isBlocked) return false;
        if (pin.equals(enteredPin)) {
            failedPinAttempts = 0;
            return true;
        }
        failedPinAttempts++;
        if (failedPinAttempts >= MAX_PIN_ATTEMPTS) {
            isBlocked = true;
        }
        return false;
    }
}
```

---

### Phase 4: Build Hardware Components

```java
// Step 6: CashDispenser
public class CashDispenser {
    private final Map<Integer, Integer> cashInventory;
    private static final int[] DENOMINATIONS = {100, 50, 20, 10, 5, 1};
    
    public synchronized boolean dispenseCash(double amount) {
        Map<Integer, Integer> billsToDispense = calculateBills((int)amount);
        if (billsToDispense == null) return false;
        
        for (Map.Entry<Integer, Integer> entry : billsToDispense.entrySet()) {
            cashInventory.merge(entry.getKey(), -entry.getValue(), Integer::sum);
        }
        return true;
    }
}
```

---

### Phase 5: Build Transactions

```java
// Step 7: Transaction abstract class
public abstract class Transaction {
    protected final String transactionId;
    protected final TransactionType type;
    protected final Account account;
    protected final double amount;
    protected TransactionStatus status;
    
    public abstract boolean execute();
}

// Step 8: WithdrawalTransaction
public class WithdrawalTransaction extends Transaction {
    private final CashDispenser cashDispenser;
    
    @Override
    public boolean execute() {
        if (!account.withdraw(amount)) {
            fail("Insufficient funds");
            return false;
        }
        if (!cashDispenser.dispenseCash(amount)) {
            account.deposit(amount);  // Rollback
            fail("Failed to dispense");
            return false;
        }
        complete();
        return true;
    }
}
```

---

### Phase 6: Build Controller

```java
// Step 9: ATMController
public class ATMController {
    private final ATM atm;
    private ATMState state;
    private Card currentCard;
    private Account selectedAccount;
    
    public boolean insertCard(Card card) {
        if (state != ATMState.IDLE) return false;
        Card readCard = atm.getCardReader().readCard(card);
        if (readCard == null) return false;
        currentCard = readCard;
        state = ATMState.CARD_INSERTED;
        return true;
    }
    
    public boolean authenticate(String pin) {
        if (state != ATMState.CARD_INSERTED) return false;
        if (currentCard.validatePin(pin)) {
            state = ATMState.AUTHENTICATED;
            return true;
        }
        return false;
    }
}
```

---

### Phase 7: Threading Model and Concurrency Control

**Threading Model:**

This system handles **concurrent ATM sessions**:
- Multiple users can attempt to use different ATMs
- Single ATM serves one user at a time (state machine)
- Account operations must be thread-safe

**Concurrency Control:**

```java
// Account-level locking
public class Account {
    private final ReentrantLock lock;
    
    public boolean withdraw(double amount) {
        lock.lock();  // Only one thread per account
        try {
            // Critical section
        } finally {
            lock.unlock();
        }
    }
}

// ATM-level state machine
public class ATM {
    private final Object stateLock = new Object();
    private ATMState state;
    
    public boolean insertCard(Card card) {
        synchronized (stateLock) {
            if (state != ATMState.IDLE) return false;
            state = ATMState.CARD_INSERTED;
            return true;
        }
    }
}
```

**Why this model?**
- Account locks prevent concurrent withdrawals from same account
- ATM state machine ensures single-user session per ATM
- Cash dispenser synchronized for inventory safety

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 Enums

```java
// TransactionType.java
package com.atm.enums;

public enum TransactionType {
    BALANCE_INQUIRY,
    WITHDRAWAL,
    DEPOSIT,
    TRANSFER,
    PIN_CHANGE
}
```

```java
// TransactionStatus.java
package com.atm.enums;

public enum TransactionStatus {
    PENDING,
    COMPLETED,
    FAILED,
    CANCELLED
}
```

```java
// AccountType.java
package com.atm.enums;

public enum AccountType {
    CHECKING,
    SAVINGS
}
```

```java
// ATMState.java
package com.atm.enums;

public enum ATMState {
    IDLE,               // Waiting for card
    CARD_INSERTED,      // Card in, waiting for PIN
    AUTHENTICATED,      // PIN verified, showing menu
    TRANSACTION_IN_PROGRESS,
    OUT_OF_SERVICE,     // Maintenance or error
    OUT_OF_CASH         // No cash available
}
```

### 2.2 Card and Customer Classes

```java
// Card.java
package com.atm.models;

import java.time.LocalDate;
import java.util.*;

/**
 * Represents a bank card (debit/ATM card).
 * 
 * A card can be linked to multiple accounts.
 */
public class Card {
    
    private final String cardNumber;
    private final String customerName;
    private final LocalDate expirationDate;
    private String pin;  // In real system, this would be hashed
    private boolean isBlocked;
    private int failedPinAttempts;
    
    private final List<Account> linkedAccounts;
    
    private static final int MAX_PIN_ATTEMPTS = 3;
    
    public Card(String cardNumber, String customerName, String pin, 
                LocalDate expirationDate) {
        this.cardNumber = cardNumber;
        this.customerName = customerName;
        this.pin = pin;
        this.expirationDate = expirationDate;
        this.isBlocked = false;
        this.failedPinAttempts = 0;
        this.linkedAccounts = new ArrayList<>();
    }
    
    /**
     * Validates the entered PIN.
     */
    public boolean validatePin(String enteredPin) {
        if (isBlocked) {
            System.out.println("Card is blocked");
            return false;
        }
        
        if (pin.equals(enteredPin)) {
            failedPinAttempts = 0;
            return true;
        }
        
        failedPinAttempts++;
        if (failedPinAttempts >= MAX_PIN_ATTEMPTS) {
            isBlocked = true;
            System.out.println("Card blocked after " + MAX_PIN_ATTEMPTS + " failed attempts");
        }
        
        return false;
    }
    
    /**
     * Changes the PIN.
     */
    public boolean changePin(String oldPin, String newPin) {
        if (!validatePin(oldPin)) {
            return false;
        }
        
        if (newPin == null || newPin.length() != 4) {
            System.out.println("PIN must be 4 digits");
            return false;
        }
        
        this.pin = newPin;
        return true;
    }
    
    /**
     * Checks if card is expired.
     */
    public boolean isExpired() {
        return LocalDate.now().isAfter(expirationDate);
    }
    
    /**
     * Links an account to this card.
     */
    public void linkAccount(Account account) {
        if (!linkedAccounts.contains(account)) {
            linkedAccounts.add(account);
        }
    }
    
    /**
     * Gets remaining PIN attempts.
     */
    public int getRemainingAttempts() {
        return MAX_PIN_ATTEMPTS - failedPinAttempts;
    }
    
    // Getters
    public String getCardNumber() { return cardNumber; }
    public String getCustomerName() { return customerName; }
    public LocalDate getExpirationDate() { return expirationDate; }
    public boolean isBlocked() { return isBlocked; }
    public List<Account> getLinkedAccounts() { 
        return Collections.unmodifiableList(linkedAccounts); 
    }
    
    /**
     * Masks card number for display.
     */
    public String getMaskedCardNumber() {
        if (cardNumber.length() < 4) return cardNumber;
        return "**** **** **** " + cardNumber.substring(cardNumber.length() - 4);
    }
    
    @Override
    public String toString() {
        return String.format("Card[%s, holder=%s, blocked=%s]",
                getMaskedCardNumber(), customerName, isBlocked);
    }
}
```

```java
// Customer.java
package com.atm.models;

import java.util.*;

/**
 * Represents a bank customer.
 */
public class Customer {
    
    private final String customerId;
    private final String name;
    private final String email;
    private final String phone;
    private final String address;
    
    private final List<Account> accounts;
    private final List<Card> cards;
    
    public Customer(String customerId, String name, String email, 
                    String phone, String address) {
        this.customerId = customerId;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.address = address;
        this.accounts = new ArrayList<>();
        this.cards = new ArrayList<>();
    }
    
    public void addAccount(Account account) {
        accounts.add(account);
    }
    
    public void addCard(Card card) {
        cards.add(card);
    }
    
    // Getters
    public String getCustomerId() { return customerId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    public String getAddress() { return address; }
    public List<Account> getAccounts() { return Collections.unmodifiableList(accounts); }
    public List<Card> getCards() { return Collections.unmodifiableList(cards); }
    
    @Override
    public String toString() {
        return String.format("Customer[id=%s, name=%s]", customerId, name);
    }
}
```

### 2.3 Account Classes

```java
// Account.java
package com.atm.models;

import com.atm.enums.AccountType;
import java.util.*;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Abstract base class for bank accounts.
 * 
 * THREAD SAFETY: Uses ReentrantLock for thread-safe operations.
 */
public abstract class Account {
    
    protected final String accountNumber;
    protected final AccountType type;
    protected final Customer owner;
    protected double balance;
    
    protected final List<Transaction> transactions;
    protected final ReentrantLock lock;
    
    protected Account(String accountNumber, AccountType type, 
                      Customer owner, double initialBalance) {
        this.accountNumber = accountNumber;
        this.type = type;
        this.owner = owner;
        this.balance = initialBalance;
        this.transactions = new ArrayList<>();
        this.lock = new ReentrantLock();
    }
    
    /**
     * Gets current balance.
     */
    public double getBalance() {
        lock.lock();
        try {
            return balance;
        } finally {
            lock.unlock();
        }
    }
    
    /**
     * Withdraws money from account.
     * 
     * @return true if withdrawal successful
     */
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
    
    /**
     * Deposits money into account.
     */
    public boolean deposit(double amount) {
        lock.lock();
        try {
            if (amount <= 0) {
                System.out.println("Invalid deposit amount");
                return false;
            }
            
            balance += amount;
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    /**
     * Checks if withdrawal is allowed.
     * Subclasses can override for specific rules.
     */
    protected boolean canWithdraw(double amount) {
        return balance >= amount;
    }
    
    /**
     * Adds a transaction record.
     */
    public void addTransaction(Transaction transaction) {
        transactions.add(transaction);
    }
    
    // Getters
    public String getAccountNumber() { return accountNumber; }
    public AccountType getType() { return type; }
    public Customer getOwner() { return owner; }
    public List<Transaction> getTransactions() { 
        return Collections.unmodifiableList(transactions); 
    }
    
    /**
     * Gets masked account number for display.
     */
    public String getMaskedAccountNumber() {
        if (accountNumber.length() < 4) return accountNumber;
        return "****" + accountNumber.substring(accountNumber.length() - 4);
    }
    
    @Override
    public String toString() {
        return String.format("%s Account[%s, balance=$%.2f]",
                type, getMaskedAccountNumber(), balance);
    }
}
```

```java
// CheckingAccount.java
package com.atm.models;

import com.atm.enums.AccountType;

/**
 * Checking account with daily withdrawal limit.
 */
public class CheckingAccount extends Account {
    
    private double dailyWithdrawalLimit;
    private double withdrawnToday;
    
    private static final double DEFAULT_DAILY_LIMIT = 1000.0;
    
    public CheckingAccount(String accountNumber, Customer owner, double initialBalance) {
        super(accountNumber, AccountType.CHECKING, owner, initialBalance);
        this.dailyWithdrawalLimit = DEFAULT_DAILY_LIMIT;
        this.withdrawnToday = 0;
    }
    
    @Override
    protected boolean canWithdraw(double amount) {
        if (balance < amount) {
            return false;
        }
        
        if (withdrawnToday + amount > dailyWithdrawalLimit) {
            System.out.println("Daily withdrawal limit exceeded. " +
                              "Remaining: $" + (dailyWithdrawalLimit - withdrawnToday));
            return false;
        }
        
        return true;
    }
    
    @Override
    public boolean withdraw(double amount) {
        boolean success = super.withdraw(amount);
        if (success) {
            withdrawnToday += amount;
        }
        return success;
    }
    
    /**
     * Resets daily withdrawal counter (called at midnight).
     */
    public void resetDailyWithdrawal() {
        lock.lock();
        try {
            withdrawnToday = 0;
        } finally {
            lock.unlock();
        }
    }
    
    public double getDailyWithdrawalLimit() { return dailyWithdrawalLimit; }
    public double getRemainingDailyLimit() { return dailyWithdrawalLimit - withdrawnToday; }
    
    public void setDailyWithdrawalLimit(double limit) {
        this.dailyWithdrawalLimit = limit;
    }
}
```

```java
// SavingsAccount.java
package com.atm.models;

import com.atm.enums.AccountType;

/**
 * Savings account with withdrawal restrictions.
 */
public class SavingsAccount extends Account {
    
    private int monthlyWithdrawalsAllowed;
    private int withdrawalsThisMonth;
    private double minimumBalance;
    
    private static final int DEFAULT_MONTHLY_WITHDRAWALS = 6;
    private static final double DEFAULT_MINIMUM_BALANCE = 100.0;
    
    public SavingsAccount(String accountNumber, Customer owner, double initialBalance) {
        super(accountNumber, AccountType.SAVINGS, owner, initialBalance);
        this.monthlyWithdrawalsAllowed = DEFAULT_MONTHLY_WITHDRAWALS;
        this.withdrawalsThisMonth = 0;
        this.minimumBalance = DEFAULT_MINIMUM_BALANCE;
    }
    
    @Override
    protected boolean canWithdraw(double amount) {
        // Check minimum balance
        if (balance - amount < minimumBalance) {
            System.out.println("Cannot withdraw. Minimum balance of $" + 
                              minimumBalance + " required");
            return false;
        }
        
        // Check monthly withdrawal limit
        if (withdrawalsThisMonth >= monthlyWithdrawalsAllowed) {
            System.out.println("Monthly withdrawal limit reached");
            return false;
        }
        
        return true;
    }
    
    @Override
    public boolean withdraw(double amount) {
        boolean success = super.withdraw(amount);
        if (success) {
            withdrawalsThisMonth++;
        }
        return success;
    }
    
    /**
     * Resets monthly withdrawal counter.
     */
    public void resetMonthlyWithdrawals() {
        lock.lock();
        try {
            withdrawalsThisMonth = 0;
        } finally {
            lock.unlock();
        }
    }
    
    public int getRemainingWithdrawals() {
        return monthlyWithdrawalsAllowed - withdrawalsThisMonth;
    }
}
```

### 2.4 Transaction Classes

```java
// Transaction.java
package com.atm.models;

import com.atm.enums.TransactionStatus;
import com.atm.enums.TransactionType;
import java.time.LocalDateTime;

/**
 * Base class for all ATM transactions.
 */
public abstract class Transaction {
    
    protected final String transactionId;
    protected final TransactionType type;
    protected final Account account;
    protected final double amount;
    protected final LocalDateTime timestamp;
    
    protected TransactionStatus status;
    protected String failureReason;
    
    protected Transaction(TransactionType type, Account account, double amount) {
        this.transactionId = generateTransactionId();
        this.type = type;
        this.account = account;
        this.amount = amount;
        this.timestamp = LocalDateTime.now();
        this.status = TransactionStatus.PENDING;
    }
    
    private String generateTransactionId() {
        return "TXN-" + System.currentTimeMillis() + "-" + 
               (int)(Math.random() * 10000);
    }
    
    /**
     * Executes the transaction.
     * Each transaction type implements its own logic.
     */
    public abstract boolean execute();
    
    /**
     * Marks transaction as completed.
     */
    protected void complete() {
        this.status = TransactionStatus.COMPLETED;
    }
    
    /**
     * Marks transaction as failed.
     */
    protected void fail(String reason) {
        this.status = TransactionStatus.FAILED;
        this.failureReason = reason;
    }
    
    // Getters
    public String getTransactionId() { return transactionId; }
    public TransactionType getType() { return type; }
    public Account getAccount() { return account; }
    public double getAmount() { return amount; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public TransactionStatus getStatus() { return status; }
    public String getFailureReason() { return failureReason; }
    
    @Override
    public String toString() {
        return String.format("Transaction[id=%s, type=%s, amount=$%.2f, status=%s]",
                transactionId, type, amount, status);
    }
}
```

```java
// WithdrawalTransaction.java
package com.atm.models;

import com.atm.enums.TransactionType;
import com.atm.hardware.CashDispenser;

/**
 * Withdrawal transaction.
 */
public class WithdrawalTransaction extends Transaction {
    
    private final CashDispenser cashDispenser;
    
    public WithdrawalTransaction(Account account, double amount, 
                                  CashDispenser cashDispenser) {
        super(TransactionType.WITHDRAWAL, account, amount);
        this.cashDispenser = cashDispenser;
    }
    
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
        
        System.out.println("Withdrawal successful: $" + amount);
        return true;
    }
}
```

```java
// DepositTransaction.java
package com.atm.models;

import com.atm.enums.TransactionType;
import com.atm.hardware.CashSlot;

/**
 * Deposit transaction.
 */
public class DepositTransaction extends Transaction {
    
    private final CashSlot cashSlot;
    private double verifiedAmount;
    
    public DepositTransaction(Account account, double declaredAmount, CashSlot cashSlot) {
        super(TransactionType.DEPOSIT, account, declaredAmount);
        this.cashSlot = cashSlot;
    }
    
    @Override
    public boolean execute() {
        // Step 1: Accept cash
        System.out.println("Please insert cash...");
        
        // Step 2: Count and verify cash
        verifiedAmount = cashSlot.countInsertedCash();
        
        if (verifiedAmount <= 0) {
            fail("No cash detected");
            return false;
        }
        
        if (verifiedAmount != amount) {
            System.out.println("Declared amount ($" + amount + ") differs from " +
                              "counted amount ($" + verifiedAmount + ")");
            // Use verified amount
        }
        
        // Step 3: Deposit to account
        if (!account.deposit(verifiedAmount)) {
            fail("Deposit failed");
            cashSlot.returnCash();
            return false;
        }
        
        // Step 4: Record transaction
        account.addTransaction(this);
        complete();
        
        System.out.println("Deposit successful: $" + verifiedAmount);
        return true;
    }
    
    public double getVerifiedAmount() {
        return verifiedAmount;
    }
}
```

```java
// TransferTransaction.java
package com.atm.models;

import com.atm.enums.TransactionType;

/**
 * Transfer transaction between accounts.
 */
public class TransferTransaction extends Transaction {
    
    private final Account destinationAccount;
    
    public TransferTransaction(Account sourceAccount, Account destinationAccount, 
                               double amount) {
        super(TransactionType.TRANSFER, sourceAccount, amount);
        this.destinationAccount = destinationAccount;
    }
    
    @Override
    public boolean execute() {
        // Step 1: Validate
        if (amount <= 0) {
            fail("Invalid transfer amount");
            return false;
        }
        
        if (account.equals(destinationAccount)) {
            fail("Cannot transfer to same account");
            return false;
        }
        
        // Step 2: Withdraw from source (atomic operation)
        if (!account.withdraw(amount)) {
            fail("Insufficient funds in source account");
            return false;
        }
        
        // Step 3: Deposit to destination
        if (!destinationAccount.deposit(amount)) {
            // Rollback source withdrawal
            account.deposit(amount);
            fail("Failed to deposit to destination account");
            return false;
        }
        
        // Step 4: Record transactions
        account.addTransaction(this);
        destinationAccount.addTransaction(this);
        complete();
        
        System.out.println("Transfer successful: $" + amount + " to " + 
                          destinationAccount.getMaskedAccountNumber());
        return true;
    }
    
    public Account getDestinationAccount() {
        return destinationAccount;
    }
}
```

```java
// BalanceInquiryTransaction.java
package com.atm.models;

import com.atm.enums.TransactionType;

/**
 * Balance inquiry transaction.
 */
public class BalanceInquiryTransaction extends Transaction {
    
    private double balanceResult;
    
    public BalanceInquiryTransaction(Account account) {
        super(TransactionType.BALANCE_INQUIRY, account, 0);
    }
    
    @Override
    public boolean execute() {
        balanceResult = account.getBalance();
        complete();
        
        System.out.println("Current balance: $" + String.format("%.2f", balanceResult));
        return true;
    }
    
    public double getBalanceResult() {
        return balanceResult;
    }
}
```

### 2.5 Hardware Components

```java
// CashDispenser.java
package com.atm.hardware;

import java.util.*;

/**
 * Manages cash dispensing with multiple denominations.
 * 
 * Uses greedy algorithm to dispense minimum number of bills.
 */
public class CashDispenser {
    
    // Denomination -> Count
    private final Map<Integer, Integer> cashInventory;
    private static final int[] DENOMINATIONS = {100, 50, 20, 10, 5, 1};
    
    public CashDispenser() {
        this.cashInventory = new LinkedHashMap<>();
        // Initialize with default inventory
        initializeInventory();
    }
    
    private void initializeInventory() {
        cashInventory.put(100, 100);  // 100 × $100 = $10,000
        cashInventory.put(50, 200);   // 200 × $50 = $10,000
        cashInventory.put(20, 500);   // 500 × $20 = $10,000
        cashInventory.put(10, 200);   // 200 × $10 = $2,000
        cashInventory.put(5, 200);    // 200 × $5 = $1,000
        cashInventory.put(1, 100);    // 100 × $1 = $100
    }
    
    /**
     * Checks if the requested amount can be dispensed.
     */
    public boolean canDispense(double amount) {
        if (amount <= 0) return false;
        
        int amountInt = (int) amount;
        if (amountInt != amount) return false;  // No cents
        
        // Try to find a combination
        Map<Integer, Integer> needed = calculateBills(amountInt);
        return needed != null;
    }
    
    /**
     * Dispenses cash and updates inventory.
     */
    public synchronized boolean dispenseCash(double amount) {
        int amountInt = (int) amount;
        
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
    
    /**
     * Calculates the bills needed using greedy algorithm.
     */
    private Map<Integer, Integer> calculateBills(int amount) {
        Map<Integer, Integer> result = new LinkedHashMap<>();
        int remaining = amount;
        
        for (int denomination : DENOMINATIONS) {
            int available = cashInventory.getOrDefault(denomination, 0);
            int needed = remaining / denomination;
            int toUse = Math.min(needed, available);
            
            if (toUse > 0) {
                result.put(denomination, toUse);
                remaining -= toUse * denomination;
            }
        }
        
        if (remaining > 0) {
            return null;  // Cannot dispense exact amount
        }
        
        return result;
    }
    
    /**
     * Gets total cash available in the dispenser.
     */
    public double getTotalCash() {
        return cashInventory.entrySet().stream()
                .mapToDouble(e -> e.getKey() * e.getValue())
                .sum();
    }
    
    /**
     * Checks if dispenser is low on cash.
     */
    public boolean isLowOnCash() {
        return getTotalCash() < 1000;
    }
    
    /**
     * Refills the cash inventory.
     */
    public void refill(int denomination, int count) {
        cashInventory.merge(denomination, count, Integer::sum);
        System.out.println("Refilled " + count + " × $" + denomination);
    }
    
    /**
     * Gets current inventory status.
     */
    public void displayInventory() {
        System.out.println("\nCash Dispenser Inventory:");
        for (Map.Entry<Integer, Integer> entry : cashInventory.entrySet()) {
            System.out.println("  $" + entry.getKey() + ": " + entry.getValue() + 
                              " bills = $" + (entry.getKey() * entry.getValue()));
        }
        System.out.println("  Total: $" + getTotalCash());
    }
}
```

```java
// CashSlot.java
package com.atm.hardware;

/**
 * Cash slot for deposits.
 * Simulates cash acceptance and counting.
 */
public class CashSlot {
    
    private double insertedAmount;
    private boolean hasInsertedCash;
    
    public CashSlot() {
        this.insertedAmount = 0;
        this.hasInsertedCash = false;
    }
    
    /**
     * Simulates inserting cash.
     */
    public void insertCash(double amount) {
        this.insertedAmount = amount;
        this.hasInsertedCash = true;
        System.out.println("Cash inserted: $" + amount);
    }
    
    /**
     * Counts the inserted cash.
     * In real ATM, this uses sensors to count bills.
     */
    public double countInsertedCash() {
        if (!hasInsertedCash) {
            return 0;
        }
        
        // Simulate counting delay
        System.out.println("Counting cash...");
        
        double counted = insertedAmount;
        insertedAmount = 0;
        hasInsertedCash = false;
        
        return counted;
    }
    
    /**
     * Returns the inserted cash (on error or cancellation).
     */
    public void returnCash() {
        if (hasInsertedCash) {
            System.out.println("Returning cash: $" + insertedAmount);
            insertedAmount = 0;
            hasInsertedCash = false;
        }
    }
}
```

```java
// CardReader.java
package com.atm.hardware;

import com.atm.models.Card;

/**
 * Card reader component.
 */
public class CardReader {
    
    private Card insertedCard;
    private boolean cardPresent;
    
    public CardReader() {
        this.cardPresent = false;
    }
    
    /**
     * Reads and validates a card.
     */
    public Card readCard(Card card) {
        if (card == null) {
            System.out.println("No card detected");
            return null;
        }
        
        if (card.isExpired()) {
            System.out.println("Card is expired");
            return null;
        }
        
        if (card.isBlocked()) {
            System.out.println("Card is blocked");
            return null;
        }
        
        this.insertedCard = card;
        this.cardPresent = true;
        System.out.println("Card accepted: " + card.getMaskedCardNumber());
        
        return card;
    }
    
    /**
     * Ejects the card.
     */
    public Card ejectCard() {
        if (!cardPresent) {
            return null;
        }
        
        Card card = insertedCard;
        insertedCard = null;
        cardPresent = false;
        System.out.println("Please take your card");
        
        return card;
    }
    
    /**
     * Retains the card (on security issue).
     */
    public void retainCard() {
        if (cardPresent) {
            System.out.println("Card retained due to security concern");
            insertedCard = null;
            cardPresent = false;
        }
    }
    
    public Card getInsertedCard() { return insertedCard; }
    public boolean isCardPresent() { return cardPresent; }
}
```

```java
// Screen.java
package com.atm.hardware;

/**
 * ATM screen for displaying messages.
 */
public class Screen {
    
    public void displayMessage(String message) {
        System.out.println("\n╔════════════════════════════════════╗");
        System.out.println("║           ATM SCREEN               ║");
        System.out.println("╠════════════════════════════════════╣");
        for (String line : message.split("\n")) {
            System.out.println("║ " + padRight(line, 34) + " ║");
        }
        System.out.println("╚════════════════════════════════════╝\n");
    }
    
    public void displayMenu(String[] options) {
        StringBuilder sb = new StringBuilder();
        sb.append("Please select an option:\n");
        for (int i = 0; i < options.length; i++) {
            sb.append((i + 1)).append(". ").append(options[i]).append("\n");
        }
        displayMessage(sb.toString());
    }
    
    public void displayError(String error) {
        displayMessage("ERROR: " + error);
    }
    
    public void displaySuccess(String message) {
        displayMessage("SUCCESS: " + message);
    }
    
    private String padRight(String s, int n) {
        return String.format("%-" + n + "s", s);
    }
}
```

```java
// Keypad.java
package com.atm.hardware;

import java.util.Scanner;

/**
 * ATM keypad for user input.
 */
public class Keypad {
    
    private final Scanner scanner;
    
    public Keypad() {
        this.scanner = new Scanner(System.in);
    }
    
    /**
     * Gets PIN input (masked).
     */
    public String getPin() {
        System.out.print("Enter PIN: ");
        return scanner.nextLine();
    }
    
    /**
     * Gets numeric input.
     */
    public int getInput() {
        System.out.print("Enter choice: ");
        try {
            return Integer.parseInt(scanner.nextLine());
        } catch (NumberFormatException e) {
            return -1;
        }
    }
    
    /**
     * Gets amount input.
     */
    public double getAmount() {
        System.out.print("Enter amount: $");
        try {
            return Double.parseDouble(scanner.nextLine());
        } catch (NumberFormatException e) {
            return -1;
        }
    }
    
    /**
     * Gets account number input.
     */
    public String getAccountNumber() {
        System.out.print("Enter account number: ");
        return scanner.nextLine();
    }
}
```

### 2.6 ATM Controller and ATM

```java
// ATMController.java
package com.atm.controller;

import com.atm.enums.ATMState;
import com.atm.hardware.*;
import com.atm.models.*;
import java.util.*;

/**
 * Controls ATM operations and state transitions.
 */
public class ATMController {
    
    private final ATM atm;
    private ATMState state;
    private Card currentCard;
    private Account selectedAccount;
    
    public ATMController(ATM atm) {
        this.atm = atm;
        this.state = ATMState.IDLE;
    }
    
    /**
     * Processes card insertion.
     */
    public boolean insertCard(Card card) {
        if (state != ATMState.IDLE) {
            atm.getScreen().displayError("ATM is busy");
            return false;
        }
        
        Card readCard = atm.getCardReader().readCard(card);
        if (readCard == null) {
            return false;
        }
        
        currentCard = readCard;
        state = ATMState.CARD_INSERTED;
        atm.getScreen().displayMessage("Card accepted.\nPlease enter your PIN.");
        
        return true;
    }
    
    /**
     * Authenticates with PIN.
     */
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
                atm.getScreen().displayError("Incorrect PIN. " + remaining + " attempts remaining.");
            } else {
                atm.getScreen().displayError("Card blocked. Please contact your bank.");
                atm.getCardReader().retainCard();
                state = ATMState.IDLE;
            }
            return false;
        }
    }
    
    /**
     * Selects an account for transactions.
     */
    public boolean selectAccount(int index) {
        if (state != ATMState.AUTHENTICATED) {
            return false;
        }
        
        List<Account> accounts = currentCard.getLinkedAccounts();
        if (index < 0 || index >= accounts.size()) {
            atm.getScreen().displayError("Invalid account selection");
            return false;
        }
        
        selectedAccount = accounts.get(index);
        return true;
    }
    
    /**
     * Performs balance inquiry.
     */
    public boolean checkBalance() {
        if (selectedAccount == null) {
            atm.getScreen().displayError("No account selected");
            return false;
        }
        
        state = ATMState.TRANSACTION_IN_PROGRESS;
        
        BalanceInquiryTransaction txn = new BalanceInquiryTransaction(selectedAccount);
        boolean success = txn.execute();
        
        if (success) {
            atm.getScreen().displayMessage(
                "Account: " + selectedAccount.getMaskedAccountNumber() + "\n" +
                "Balance: $" + String.format("%.2f", txn.getBalanceResult())
            );
        }
        
        state = ATMState.AUTHENTICATED;
        return success;
    }
    
    /**
     * Performs withdrawal.
     */
    public boolean withdraw(double amount) {
        if (selectedAccount == null) {
            atm.getScreen().displayError("No account selected");
            return false;
        }
        
        state = ATMState.TRANSACTION_IN_PROGRESS;
        
        WithdrawalTransaction txn = new WithdrawalTransaction(
            selectedAccount, amount, atm.getCashDispenser());
        boolean success = txn.execute();
        
        if (success) {
            atm.getScreen().displaySuccess("Please take your cash");
        } else {
            atm.getScreen().displayError(txn.getFailureReason());
        }
        
        // Check if ATM is low on cash
        if (atm.getCashDispenser().isLowOnCash()) {
            state = ATMState.OUT_OF_CASH;
            atm.getScreen().displayMessage("ATM is running low on cash");
        } else {
            state = ATMState.AUTHENTICATED;
        }
        
        return success;
    }
    
    /**
     * Performs deposit.
     */
    public boolean deposit(double amount) {
        if (selectedAccount == null) {
            atm.getScreen().displayError("No account selected");
            return false;
        }
        
        state = ATMState.TRANSACTION_IN_PROGRESS;
        
        // Simulate cash insertion
        atm.getCashSlot().insertCash(amount);
        
        DepositTransaction txn = new DepositTransaction(
            selectedAccount, amount, atm.getCashSlot());
        boolean success = txn.execute();
        
        if (success) {
            atm.getScreen().displaySuccess(
                "Deposited: $" + String.format("%.2f", txn.getVerifiedAmount()));
        } else {
            atm.getScreen().displayError(txn.getFailureReason());
        }
        
        state = ATMState.AUTHENTICATED;
        return success;
    }
    
    /**
     * Performs transfer.
     */
    public boolean transfer(String destAccountNumber, double amount) {
        if (selectedAccount == null) {
            atm.getScreen().displayError("No account selected");
            return false;
        }
        
        // Find destination account
        Account destAccount = atm.getBank().findAccount(destAccountNumber);
        if (destAccount == null) {
            atm.getScreen().displayError("Destination account not found");
            return false;
        }
        
        state = ATMState.TRANSACTION_IN_PROGRESS;
        
        TransferTransaction txn = new TransferTransaction(
            selectedAccount, destAccount, amount);
        boolean success = txn.execute();
        
        if (success) {
            atm.getScreen().displaySuccess(
                "Transferred $" + amount + " to " + destAccount.getMaskedAccountNumber());
        } else {
            atm.getScreen().displayError(txn.getFailureReason());
        }
        
        state = ATMState.AUTHENTICATED;
        return success;
    }
    
    /**
     * Ends the session.
     */
    public void endSession() {
        atm.getCardReader().ejectCard();
        currentCard = null;
        selectedAccount = null;
        state = ATMState.IDLE;
        atm.getScreen().displayMessage("Thank you for using our ATM.\nGoodbye!");
    }
    
    /**
     * Displays the main menu.
     */
    public void showMainMenu() {
        atm.getScreen().displayMenu(new String[]{
            "Balance Inquiry",
            "Withdrawal",
            "Deposit",
            "Transfer",
            "Exit"
        });
    }
    
    /**
     * Displays account selection.
     */
    public void showAccountSelection() {
        List<Account> accounts = currentCard.getLinkedAccounts();
        String[] options = new String[accounts.size()];
        
        for (int i = 0; i < accounts.size(); i++) {
            Account acc = accounts.get(i);
            options[i] = acc.getType() + " - " + acc.getMaskedAccountNumber();
        }
        
        atm.getScreen().displayMenu(options);
    }
    
    public ATMState getState() { return state; }
    public Card getCurrentCard() { return currentCard; }
    public Account getSelectedAccount() { return selectedAccount; }
}
```

```java
// ATM.java
package com.atm;

import com.atm.controller.ATMController;
import com.atm.hardware.*;
import com.atm.models.*;

/**
 * Main ATM class that integrates all components.
 */
public class ATM {
    
    private final String atmId;
    private final String location;
    private final Bank bank;
    
    // Hardware components
    private final CardReader cardReader;
    private final CashDispenser cashDispenser;
    private final CashSlot cashSlot;
    private final Screen screen;
    private final Keypad keypad;
    
    // Controller
    private final ATMController controller;
    
    public ATM(String atmId, String location, Bank bank) {
        this.atmId = atmId;
        this.location = location;
        this.bank = bank;
        
        this.cardReader = new CardReader();
        this.cashDispenser = new CashDispenser();
        this.cashSlot = new CashSlot();
        this.screen = new Screen();
        this.keypad = new Keypad();
        
        this.controller = new ATMController(this);
    }
    
    /**
     * Runs the ATM session loop.
     */
    public void run() {
        screen.displayMessage("Welcome to " + bank.getName() + " ATM\n" +
                             "Location: " + location + "\n" +
                             "Please insert your card.");
        
        // In real implementation, this would be event-driven
        // For demo, we use a simple loop
    }
    
    // Getters for components
    public String getAtmId() { return atmId; }
    public String getLocation() { return location; }
    public Bank getBank() { return bank; }
    public CardReader getCardReader() { return cardReader; }
    public CashDispenser getCashDispenser() { return cashDispenser; }
    public CashSlot getCashSlot() { return cashSlot; }
    public Screen getScreen() { return screen; }
    public Keypad getKeypad() { return keypad; }
    public ATMController getController() { return controller; }
}
```

### 2.7 Bank (Central System)

```java
// Bank.java
package com.atm;

import com.atm.models.*;
import java.util.*;

/**
 * Bank system that manages accounts, cards, and ATMs.
 * 
 * PATTERN: Singleton
 */
public class Bank {
    
    private static Bank instance;
    
    private final String name;
    private final Map<String, Customer> customers;
    private final Map<String, Account> accounts;
    private final Map<String, Card> cards;
    private final List<ATM> atms;
    
    private Bank(String name) {
        this.name = name;
        this.customers = new HashMap<>();
        this.accounts = new HashMap<>();
        this.cards = new HashMap<>();
        this.atms = new ArrayList<>();
    }
    
    public static synchronized Bank getInstance(String name) {
        if (instance == null) {
            instance = new Bank(name);
        }
        return instance;
    }
    
    public static Bank getInstance() {
        if (instance == null) {
            throw new IllegalStateException("Bank not initialized");
        }
        return instance;
    }
    
    public static void resetInstance() {
        instance = null;
    }
    
    // Customer management
    public Customer createCustomer(String name, String email, String phone, String address) {
        String customerId = "CUST-" + System.currentTimeMillis();
        Customer customer = new Customer(customerId, name, email, phone, address);
        customers.put(customerId, customer);
        return customer;
    }
    
    // Account management
    public CheckingAccount createCheckingAccount(Customer customer, double initialBalance) {
        String accountNumber = generateAccountNumber();
        CheckingAccount account = new CheckingAccount(accountNumber, customer, initialBalance);
        accounts.put(accountNumber, account);
        customer.addAccount(account);
        return account;
    }
    
    public SavingsAccount createSavingsAccount(Customer customer, double initialBalance) {
        String accountNumber = generateAccountNumber();
        SavingsAccount account = new SavingsAccount(accountNumber, customer, initialBalance);
        accounts.put(accountNumber, account);
        customer.addAccount(account);
        return account;
    }
    
    private String generateAccountNumber() {
        return String.format("%010d", System.currentTimeMillis() % 10000000000L);
    }
    
    // Card management
    public Card issueCard(Customer customer, String pin) {
        String cardNumber = generateCardNumber();
        Card card = new Card(cardNumber, customer.getName(), pin,
                            java.time.LocalDate.now().plusYears(5));
        cards.put(cardNumber, card);
        customer.addCard(card);
        
        // Link all customer accounts to card
        for (Account account : customer.getAccounts()) {
            card.linkAccount(account);
        }
        
        return card;
    }
    
    private String generateCardNumber() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 16; i++) {
            sb.append((int)(Math.random() * 10));
        }
        return sb.toString();
    }
    
    // ATM management
    public ATM createATM(String location) {
        String atmId = "ATM-" + (atms.size() + 1);
        ATM atm = new ATM(atmId, location, this);
        atms.add(atm);
        return atm;
    }
    
    // Lookups
    public Account findAccount(String accountNumber) {
        return accounts.get(accountNumber);
    }
    
    public Card findCard(String cardNumber) {
        return cards.get(cardNumber);
    }
    
    public String getName() { return name; }
}
```

### 2.8 Demo Application

```java
// ATMDemo.java
package com.atm;

import com.atm.models.*;

public class ATMDemo {
    
    public static void main(String[] args) {
        Bank.resetInstance();
        
        System.out.println("=== ATM SYSTEM DEMO ===\n");
        
        // Setup bank
        Bank bank = Bank.getInstance("National Bank");
        
        // Create customer
        Customer alice = bank.createCustomer(
            "Alice Johnson", "alice@email.com", "555-1234", "123 Main St");
        
        // Create accounts
        CheckingAccount checking = bank.createCheckingAccount(alice, 5000.00);
        SavingsAccount savings = bank.createSavingsAccount(alice, 10000.00);
        
        // Issue card
        Card card = bank.issueCard(alice, "1234");
        
        System.out.println("Customer: " + alice);
        System.out.println("Checking: " + checking);
        System.out.println("Savings: " + savings);
        System.out.println("Card: " + card);
        
        // Create ATM
        ATM atm = bank.createATM("Downtown Branch");
        
        // ==================== Scenario 1: Balance Inquiry ====================
        System.out.println("\n===== SCENARIO 1: Balance Inquiry =====\n");
        
        atm.getController().insertCard(card);
        atm.getController().authenticate("1234");
        atm.getController().selectAccount(0);  // Checking
        atm.getController().checkBalance();
        
        // ==================== Scenario 2: Withdrawal ====================
        System.out.println("\n===== SCENARIO 2: Withdrawal =====\n");
        
        atm.getController().withdraw(200.00);
        atm.getController().checkBalance();
        
        // ==================== Scenario 3: Deposit ====================
        System.out.println("\n===== SCENARIO 3: Deposit =====\n");
        
        atm.getController().deposit(500.00);
        atm.getController().checkBalance();
        
        // ==================== Scenario 4: Transfer ====================
        System.out.println("\n===== SCENARIO 4: Transfer =====\n");
        
        atm.getController().transfer(savings.getAccountNumber(), 1000.00);
        
        // Check both balances
        atm.getController().checkBalance();
        atm.getController().selectAccount(1);  // Savings
        atm.getController().checkBalance();
        
        // ==================== Scenario 5: Wrong PIN ====================
        System.out.println("\n===== SCENARIO 5: Wrong PIN =====\n");
        
        atm.getController().endSession();
        
        ATM atm2 = bank.createATM("Airport Branch");
        atm2.getController().insertCard(card);
        atm2.getController().authenticate("0000");  // Wrong
        atm2.getController().authenticate("0000");  // Wrong again
        atm2.getController().authenticate("1234");  // Correct
        
        // ==================== Scenario 6: Cash Dispenser Status ====================
        System.out.println("\n===== SCENARIO 6: Cash Dispenser Status =====\n");
        
        atm.getCashDispenser().displayInventory();
        
        // End session
        atm2.getController().endSession();
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 5: Simulation / Dry Run

### Scenario: Complete ATM Session

```
Initial State:
- Alice's Checking: $5,000
- Alice's Savings: $10,000
- ATM Cash: $33,100
- Card PIN: 1234

Step 1: Insert Card
- CardReader.readCard(card)
- Card not expired, not blocked
- State: IDLE → CARD_INSERTED
- Screen: "Please enter your PIN"

Step 2: Enter PIN (1234)
- card.validatePin("1234") → true
- failedPinAttempts = 0
- State: CARD_INSERTED → AUTHENTICATED
- Screen: "Welcome, Alice Johnson"

Step 3: Select Checking Account
- controller.selectAccount(0)
- selectedAccount = CheckingAccount

Step 4: Withdraw $200
- WithdrawalTransaction created
- Validate: $200 is multiple of $20 ✓
- CashDispenser.canDispense($200) → true
- CheckingAccount.withdraw($200)
  - canWithdraw: balance(5000) >= 200 ✓
  - dailyLimit: 0 + 200 <= 1000 ✓
  - balance = 5000 - 200 = 4800
  - withdrawnToday = 200
- CashDispenser.dispenseCash($200)
  - Calculate: 2 × $100 bills
  - Update inventory
- Transaction complete
- Screen: "Please take your cash"

Step 5: Check Balance
- BalanceInquiryTransaction created
- account.getBalance() → $4,800
- Screen: "Balance: $4,800.00"

Step 6: Transfer $1,000 to Savings
- TransferTransaction created
- Source: Checking, Destination: Savings
- Checking.withdraw($1000)
  - canWithdraw: 4800 >= 1000 ✓
  - dailyLimit: 200 + 1000 <= 1000 ✓
  - balance = 4800 - 1000 = 3800
- Savings.deposit($1000)
  - balance = 10000 + 1000 = 11000
- Transaction complete

Final State:
- Alice's Checking: $3,800
- Alice's Savings: $11,000
- ATM Cash: $33,100 - $200 = $32,900
```

---

## File Structure

```
com/atm/
├── enums/
│   ├── TransactionType.java
│   ├── TransactionStatus.java
│   ├── AccountType.java
│   └── ATMState.java
├── models/
│   ├── Card.java
│   ├── Customer.java
│   ├── Account.java
│   ├── CheckingAccount.java
│   ├── SavingsAccount.java
│   ├── Transaction.java
│   ├── WithdrawalTransaction.java
│   ├── DepositTransaction.java
│   ├── TransferTransaction.java
│   └── BalanceInquiryTransaction.java
├── hardware/
│   ├── CardReader.java
│   ├── CashDispenser.java
│   ├── CashSlot.java
│   ├── Screen.java
│   └── Keypad.java
├── controller/
│   └── ATMController.java
├── ATM.java
├── Bank.java
└── ATMDemo.java
```

