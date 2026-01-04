# 💸 Splitwise (Expense Sharing) - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements
- Create groups and add members
- Add expenses with different split strategies (equal, exact, percentage)
- Track balances between users (who owes whom)
- Simplify debts to minimize transactions
- Show expense history and settlements
- Record payments/settlements between users

### Explicit Out-of-Scope Items
- Payment processing
- Currency conversion
- Recurring expenses
- Receipt scanning
- Bank account integration
- Notifications

### Assumptions and Constraints
- **Single Currency**: All amounts in same currency
- **Group-Based**: Expenses within groups
- **Split Types**: Equal, Exact amounts, Percentage
- **No Negative Amounts**: All expenses positive

### Public APIs
- `createGroup(name, members)`: Create expense group
- `addExpense(group, paidBy, amount, splitType, splits)`: Add expense
- `getBalances(userId)`: Get all balances for user
- `getGroupBalances(groupId)`: Get balances in group
- `settleUp(from, to, amount)`: Record payment
- `simplifyDebts(groupId)`: Minimize transactions

### Public API Usage Examples
```java
// Example 1: Basic usage - Create group and add expense
ExpenseService service = new ExpenseService();
User alice = service.createUser("Alice", "alice@email.com");
User bob = service.createUser("Bob", "bob@email.com");
Group group = service.createGroup("Trip", Arrays.asList(alice.getId(), bob.getId()));

List<Split> splits = Arrays.asList(
    new EqualSplit(alice.getId()),
    new EqualSplit(bob.getId()));
Expense expense = service.addExpense(group.getId(), alice.getId(), 
    new BigDecimal("100.00"), "Dinner", splits);
System.out.println(expense);

// Example 2: Typical workflow - Multiple expenses and balance check
service.addEqualExpense(group.getId(), bob.getId(), 
    new BigDecimal("60.00"), "Taxi", 
    Arrays.asList(alice.getId(), bob.getId()));
Map<String, BigDecimal> balances = service.getBalances(alice.getId());
service.printBalances(alice.getId());

// Example 3: Edge case - Debt simplification
List<Settlement> settlements = service.simplifyDebts(group.getId());
for (Settlement s : settlements) {
    System.out.println(s);
}
```

### Invariants
- **Zero Sum**: Total credits = Total debits in group
- **Balance Accuracy**: Running balance always correct
- **Split Validation**: Split amounts sum to total

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         EXPENSE SHARING SYSTEM                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                        ExpenseService                                     │   │
│  │                                                                           │   │
│  │  - users: Map<String, User>                                              │   │
│  │  - groups: Map<String, Group>                                            │   │
│  │  - expenses: Map<String, Expense>                                        │   │
│  │                                                                           │   │
│  │  + addExpense(paidBy, amount, splits, desc): Expense                     │   │
│  │  + getBalance(userId): Map<String, BigDecimal>                           │   │
│  │  + simplifyDebts(groupId): List<Settlement>                              │   │
│  │  + settleUp(fromUser, toUser, amount): void                              │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│           ┌──────────────┼──────────────┬────────────────┐                      │
│           │              │              │                │                      │
│           ▼              ▼              ▼                ▼                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │    User     │  │   Group     │  │   Expense   │  │    Split    │            │
│  │             │  │             │  │             │  │             │            │
│  │ - id        │  │ - id        │  │ - id        │  │ - userId    │            │
│  │ - name      │  │ - name      │  │ - paidBy    │  │ - amount    │            │
│  │ - email     │  │ - members[] │  │ - amount    │  │             │            │
│  │ - balances  │  │ - expenses[]│  │ - splits[]  │  └─────────────┘            │
│  └─────────────┘  └─────────────┘  │ - type      │         ▲                   │
│                                    └─────────────┘         │                   │
│                                                            │                   │
│                          ┌─────────────────────────────────┼─────────┐         │
│                          │                                 │         │         │
│                   ┌──────┴──────┐  ┌───────────────┐  ┌───┴────────┐│         │
│                   │ EqualSplit  │  │  ExactSplit   │  │PercentSplit││         │
│                   └─────────────┘  └───────────────┘  └────────────┘│         │
│                                                                      │         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Debt Simplification Algorithm

```
Before Simplification:
┌──────────────────────────────────────────┐
│  Alice ──$20──► Bob                      │
│  Bob ──$15──► Charlie                    │
│  Charlie ──$10──► Alice                  │
│                                          │
│  Total: 3 transactions                   │
└──────────────────────────────────────────┘

After Simplification:
┌──────────────────────────────────────────┐
│  Net Balances:                           │
│    Alice: -20 + 10 = -10 (owes)         │
│    Bob: +20 - 15 = +5 (owed)            │
│    Charlie: +15 - 10 = +5 (owed)        │
│                                          │
│  Simplified:                             │
│    Alice ──$5──► Bob                     │
│    Alice ──$5──► Charlie                 │
│                                          │
│  Total: 2 transactions                   │
└──────────────────────────────────────────┘
```

---

### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `User` | User data and balance tracking with other users | Stores user information and maintains balances - encapsulates user-level balance tracking |
| `Group` | Group membership and expense tracking | Manages group configuration - encapsulates group membership and group-level operations |
| `Expense` | Expense details and split calculation | Encapsulates expense data - stores expense information and calculates split amounts |
| `Split` (abstract) | Split amount definition contract | Base class for splits - enables multiple split types (equal, exact, percentage) |
| `EqualSplit` | Equal split calculation | Implements equal splitting - separate class for equal split logic |
| `ExactSplit` | Exact amount split | Implements exact amount splitting - separate class for exact split logic |
| `PercentSplit` | Percentage-based split | Implements percentage splitting - separate class for percentage split logic |
| `Settlement` | Debt settlement representation | Represents a settlement transaction - encapsulates settlement information for debt simplification |
| `ExpenseService` | Expense operations coordination | Coordinates expense workflow - separates business logic from domain objects, handles add expense/simplify debts |

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 SplitType Enum

```java
// SplitType.java
package com.splitwise;

public enum SplitType {
    EQUAL,      // Split equally among all
    EXACT,      // Exact amounts specified
    PERCENTAGE  // Percentage-based split
}
```

### 2.2 Split Classes

```java
// Split.java
package com.splitwise;

import java.math.BigDecimal;

/**
 * Base class for expense splits.
 */
public abstract class Split {
    
    protected final String userId;
    protected BigDecimal amount;
    
    public Split(String userId) {
        this.userId = userId;
    }
    
    public String getUserId() { return userId; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    
    public abstract SplitType getType();
}
```

```java
// EqualSplit.java
package com.splitwise;

public class EqualSplit extends Split {
    
    public EqualSplit(String userId) {
        super(userId);
    }
    
    @Override
    public SplitType getType() {
        return SplitType.EQUAL;
    }
}
```

```java
// ExactSplit.java
package com.splitwise;

import java.math.BigDecimal;

public class ExactSplit extends Split {
    
    public ExactSplit(String userId, BigDecimal amount) {
        super(userId);
        this.amount = amount;
    }
    
    @Override
    public SplitType getType() {
        return SplitType.EXACT;
    }
}
```

```java
// PercentSplit.java
package com.splitwise;

import java.math.BigDecimal;

public class PercentSplit extends Split {
    
    private final BigDecimal percentage;
    
    public PercentSplit(String userId, BigDecimal percentage) {
        super(userId);
        this.percentage = percentage;
    }
    
    public BigDecimal getPercentage() { return percentage; }
    
    @Override
    public SplitType getType() {
        return SplitType.PERCENTAGE;
    }
}
```

### 2.3 User Class

```java
// User.java
package com.splitwise;

import java.math.BigDecimal;
import java.util.*;

/**
 * Represents a user in the expense sharing system.
 */
public class User {
    
    private final String id;
    private final String name;
    private final String email;
    
    // Balance with other users: positive = they owe me, negative = I owe them
    private final Map<String, BigDecimal> balances;
    
    public User(String name, String email) {
        this.id = "USR-" + System.currentTimeMillis() % 100000;
        this.name = name;
        this.email = email;
        this.balances = new HashMap<>();
    }
    
    public void updateBalance(String otherUserId, BigDecimal amount) {
        balances.merge(otherUserId, amount, BigDecimal::add);
        
        // Remove zero balances
        if (balances.get(otherUserId).compareTo(BigDecimal.ZERO) == 0) {
            balances.remove(otherUserId);
        }
    }
    
    public BigDecimal getBalanceWith(String otherUserId) {
        return balances.getOrDefault(otherUserId, BigDecimal.ZERO);
    }
    
    public BigDecimal getTotalOwed() {
        return balances.values().stream()
            .filter(b -> b.compareTo(BigDecimal.ZERO) > 0)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    public BigDecimal getTotalOwing() {
        return balances.values().stream()
            .filter(b -> b.compareTo(BigDecimal.ZERO) < 0)
            .map(BigDecimal::abs)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    public Map<String, BigDecimal> getBalances() {
        return Collections.unmodifiableMap(balances);
    }
    
    @Override
    public String toString() {
        return String.format("%s (%s)", name, email);
    }
}
```

### 2.4 Expense Class

```java
// Expense.java
package com.splitwise;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Represents an expense in the system.
 */
public class Expense {
    
    private final String id;
    private final String description;
    private final BigDecimal amount;
    private final String paidBy;
    private final List<Split> splits;
    private final SplitType splitType;
    private final LocalDateTime timestamp;
    private final String groupId;
    
    public Expense(String description, BigDecimal amount, String paidBy,
                   List<Split> splits, String groupId) {
        this.id = "EXP-" + System.currentTimeMillis() % 100000;
        this.description = description;
        this.amount = amount;
        this.paidBy = paidBy;
        this.splits = new ArrayList<>(splits);
        this.splitType = splits.isEmpty() ? SplitType.EQUAL : splits.get(0).getType();
        this.timestamp = LocalDateTime.now();
        this.groupId = groupId;
        
        calculateSplitAmounts();
        validateSplits();
    }
    
    private void calculateSplitAmounts() {
        if (splitType == SplitType.EQUAL) {
            BigDecimal splitAmount = amount.divide(
                BigDecimal.valueOf(splits.size()), 2, RoundingMode.HALF_UP);
            
            // Handle rounding - give extra cent to first person
            BigDecimal total = splitAmount.multiply(BigDecimal.valueOf(splits.size()));
            BigDecimal diff = amount.subtract(total);
            
            for (int i = 0; i < splits.size(); i++) {
                if (i == 0) {
                    splits.get(i).setAmount(splitAmount.add(diff));
                } else {
                    splits.get(i).setAmount(splitAmount);
                }
            }
        } else if (splitType == SplitType.PERCENTAGE) {
            for (Split split : splits) {
                PercentSplit ps = (PercentSplit) split;
                BigDecimal splitAmount = amount.multiply(ps.getPercentage())
                    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
                split.setAmount(splitAmount);
            }
        }
        // EXACT splits already have amounts set
    }
    
    private void validateSplits() {
        BigDecimal totalSplit = splits.stream()
            .map(Split::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        if (totalSplit.compareTo(amount) != 0) {
            throw new IllegalArgumentException(
                "Split amounts don't add up to total: " + totalSplit + " vs " + amount);
        }
    }
    
    // Getters
    public String getId() { return id; }
    public String getDescription() { return description; }
    public BigDecimal getAmount() { return amount; }
    public String getPaidBy() { return paidBy; }
    public List<Split> getSplits() { return Collections.unmodifiableList(splits); }
    public SplitType getSplitType() { return splitType; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public String getGroupId() { return groupId; }
    
    @Override
    public String toString() {
        return String.format("%s: $%.2f paid by %s (%s split)",
            description, amount, paidBy, splitType);
    }
}
```

### 2.5 Group Class

```java
// Group.java
package com.splitwise;

import java.util.*;

/**
 * Represents a group for expense sharing.
 */
public class Group {
    
    private final String id;
    private final String name;
    private final Set<String> memberIds;
    private final List<String> expenseIds;
    
    public Group(String name) {
        this.id = "GRP-" + System.currentTimeMillis() % 100000;
        this.name = name;
        this.memberIds = new HashSet<>();
        this.expenseIds = new ArrayList<>();
    }
    
    public void addMember(String userId) {
        memberIds.add(userId);
    }
    
    public void removeMember(String userId) {
        memberIds.remove(userId);
    }
    
    public void addExpense(String expenseId) {
        expenseIds.add(0, expenseId);  // Most recent first
    }
    
    public boolean hasMember(String userId) {
        return memberIds.contains(userId);
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    
    public Set<String> getMemberIds() {
        return Collections.unmodifiableSet(memberIds);
    }
    
    public List<String> getExpenseIds() {
        return Collections.unmodifiableList(expenseIds);
    }
    
    @Override
    public String toString() {
        return String.format("%s (%d members)", name, memberIds.size());
    }
}
```

### 2.6 Settlement Class

```java
// Settlement.java
package com.splitwise;

import java.math.BigDecimal;

/**
 * Represents a settlement suggestion.
 */
public class Settlement {
    
    private final String fromUserId;
    private final String toUserId;
    private final BigDecimal amount;
    
    public Settlement(String fromUserId, String toUserId, BigDecimal amount) {
        this.fromUserId = fromUserId;
        this.toUserId = toUserId;
        this.amount = amount;
    }
    
    public String getFromUserId() { return fromUserId; }
    public String getToUserId() { return toUserId; }
    public BigDecimal getAmount() { return amount; }
    
    @Override
    public String toString() {
        return String.format("%s pays %s: $%.2f", fromUserId, toUserId, amount);
    }
}
```

### 2.7 ExpenseService Class

```java
// ExpenseService.java
package com.splitwise;

import java.math.BigDecimal;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Main service for expense sharing operations.
 */
public class ExpenseService {
    
    private final Map<String, User> users;
    private final Map<String, Group> groups;
    private final Map<String, Expense> expenses;
    
    public ExpenseService() {
        this.users = new HashMap<>();
        this.groups = new HashMap<>();
        this.expenses = new HashMap<>();
    }
    
    // ==================== User Management ====================
    
    public User createUser(String name, String email) {
        User user = new User(name, email);
        users.put(user.getId(), user);
        return user;
    }
    
    public User getUser(String userId) {
        User user = users.get(userId);
        if (user == null) {
            throw new IllegalArgumentException("User not found: " + userId);
        }
        return user;
    }
    
    // ==================== Group Management ====================
    
    public Group createGroup(String name, List<String> memberIds) {
        Group group = new Group(name);
        for (String memberId : memberIds) {
            getUser(memberId);  // Validate user exists
            group.addMember(memberId);
        }
        groups.put(group.getId(), group);
        return group;
    }
    
    public void addMemberToGroup(String groupId, String userId) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        getUser(userId);
        group.addMember(userId);
    }
    
    // ==================== Expense Management ====================
    
    public Expense addExpense(String groupId, String paidBy, BigDecimal amount,
                             String description, List<Split> splits) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        
        // Validate payer is in group
        if (!group.hasMember(paidBy)) {
            throw new IllegalArgumentException("Payer not in group");
        }
        
        // Validate all split users are in group
        for (Split split : splits) {
            if (!group.hasMember(split.getUserId())) {
                throw new IllegalArgumentException(
                    "Split user not in group: " + split.getUserId());
            }
        }
        
        Expense expense = new Expense(description, amount, paidBy, splits, groupId);
        expenses.put(expense.getId(), expense);
        group.addExpense(expense.getId());
        
        // Update balances
        updateBalances(expense);
        
        return expense;
    }
    
    public Expense addEqualExpense(String groupId, String paidBy, BigDecimal amount,
                                   String description, List<String> participantIds) {
        List<Split> splits = participantIds.stream()
            .map(EqualSplit::new)
            .collect(Collectors.toList());
        return addExpense(groupId, paidBy, amount, description, splits);
    }
    
    private void updateBalances(Expense expense) {
        String paidBy = expense.getPaidBy();
        User payer = getUser(paidBy);
        
        for (Split split : expense.getSplits()) {
            String owerId = split.getUserId();
            BigDecimal owedAmount = split.getAmount();
            
            if (!owerId.equals(paidBy)) {
                // Payer is owed this amount by ower
                payer.updateBalance(owerId, owedAmount);
                
                // Ower owes this amount to payer
                User ower = getUser(owerId);
                ower.updateBalance(paidBy, owedAmount.negate());
            }
        }
    }
    
    // ==================== Balance Queries ====================
    
    public Map<String, BigDecimal> getBalances(String userId) {
        return getUser(userId).getBalances();
    }
    
    public BigDecimal getBalance(String userId1, String userId2) {
        return getUser(userId1).getBalanceWith(userId2);
    }
    
    public void printBalances(String userId) {
        User user = getUser(userId);
        Map<String, BigDecimal> balances = user.getBalances();
        
        System.out.println("\nBalances for " + user.getName() + ":");
        
        if (balances.isEmpty()) {
            System.out.println("  All settled up!");
            return;
        }
        
        for (Map.Entry<String, BigDecimal> entry : balances.entrySet()) {
            User other = getUser(entry.getKey());
            BigDecimal amount = entry.getValue();
            
            if (amount.compareTo(BigDecimal.ZERO) > 0) {
                System.out.printf("  %s owes you $%.2f%n", other.getName(), amount);
            } else {
                System.out.printf("  You owe %s $%.2f%n", other.getName(), amount.abs());
            }
        }
    }
    
    // ==================== Settlement ====================
    
    public void settleUp(String fromUserId, String toUserId, BigDecimal amount) {
        User from = getUser(fromUserId);
        User to = getUser(toUserId);
        
        BigDecimal currentBalance = from.getBalanceWith(toUserId);
        
        if (currentBalance.compareTo(BigDecimal.ZERO) >= 0) {
            throw new IllegalStateException(
                from.getName() + " doesn't owe " + to.getName() + " anything");
        }
        
        if (amount.compareTo(currentBalance.abs()) > 0) {
            throw new IllegalArgumentException("Settlement amount exceeds debt");
        }
        
        from.updateBalance(toUserId, amount);
        to.updateBalance(fromUserId, amount.negate());
    }
    
    // ==================== Debt Simplification ====================
    
    /**
     * Simplifies debts within a group to minimize transactions.
     * Uses a greedy algorithm to match creditors with debtors.
     */
    public List<Settlement> simplifyDebts(String groupId) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        
        // Calculate net balance for each member
        Map<String, BigDecimal> netBalances = new HashMap<>();
        
        for (String memberId : group.getMemberIds()) {
            BigDecimal net = BigDecimal.ZERO;
            User user = getUser(memberId);
            
            for (String otherId : group.getMemberIds()) {
                if (!otherId.equals(memberId)) {
                    net = net.add(user.getBalanceWith(otherId));
                }
            }
            
            if (net.compareTo(BigDecimal.ZERO) != 0) {
                netBalances.put(memberId, net);
            }
        }
        
        // Separate into creditors (positive) and debtors (negative)
        PriorityQueue<Map.Entry<String, BigDecimal>> creditors = 
            new PriorityQueue<>((a, b) -> b.getValue().compareTo(a.getValue()));
        PriorityQueue<Map.Entry<String, BigDecimal>> debtors = 
            new PriorityQueue<>((a, b) -> a.getValue().compareTo(b.getValue()));
        
        for (Map.Entry<String, BigDecimal> entry : netBalances.entrySet()) {
            if (entry.getValue().compareTo(BigDecimal.ZERO) > 0) {
                creditors.offer(entry);
            } else if (entry.getValue().compareTo(BigDecimal.ZERO) < 0) {
                debtors.offer(entry);
            }
        }
        
        // Match creditors with debtors
        List<Settlement> settlements = new ArrayList<>();
        
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            Map.Entry<String, BigDecimal> creditor = creditors.poll();
            Map.Entry<String, BigDecimal> debtor = debtors.poll();
            
            BigDecimal credit = creditor.getValue();
            BigDecimal debt = debtor.getValue().abs();
            
            BigDecimal settlementAmount = credit.min(debt);
            
            settlements.add(new Settlement(
                debtor.getKey(), creditor.getKey(), settlementAmount));
            
            // Update remaining balances
            BigDecimal remainingCredit = credit.subtract(settlementAmount);
            BigDecimal remainingDebt = debt.subtract(settlementAmount);
            
            if (remainingCredit.compareTo(BigDecimal.ZERO) > 0) {
                creditors.offer(new AbstractMap.SimpleEntry<>(
                    creditor.getKey(), remainingCredit));
            }
            
            if (remainingDebt.compareTo(BigDecimal.ZERO) > 0) {
                debtors.offer(new AbstractMap.SimpleEntry<>(
                    debtor.getKey(), remainingDebt.negate()));
            }
        }
        
        return settlements;
    }
    
    // ==================== Expense History ====================
    
    public List<Expense> getGroupExpenses(String groupId) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        
        return group.getExpenseIds().stream()
            .map(expenses::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
    
    public List<Expense> getUserExpenses(String userId) {
        getUser(userId);
        
        return expenses.values().stream()
            .filter(e -> e.getPaidBy().equals(userId) ||
                        e.getSplits().stream()
                            .anyMatch(s -> s.getUserId().equals(userId)))
            .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
            .collect(Collectors.toList());
    }
}
```

### 2.8 Demo Application

```java
// SplitwiseDemo.java
package com.splitwise;

import java.math.BigDecimal;
import java.util.*;

public class SplitwiseDemo {
    
    public static void main(String[] args) {
        System.out.println("=== SPLITWISE EXPENSE SHARING DEMO ===\n");
        
        ExpenseService service = new ExpenseService();
        
        // ==================== Create Users ====================
        System.out.println("===== CREATING USERS =====\n");
        
        User alice = service.createUser("Alice", "alice@email.com");
        User bob = service.createUser("Bob", "bob@email.com");
        User charlie = service.createUser("Charlie", "charlie@email.com");
        User diana = service.createUser("Diana", "diana@email.com");
        
        System.out.println("Created: " + alice);
        System.out.println("Created: " + bob);
        System.out.println("Created: " + charlie);
        System.out.println("Created: " + diana);
        
        // ==================== Create Group ====================
        System.out.println("\n===== CREATING GROUP =====\n");
        
        Group tripGroup = service.createGroup("Beach Trip",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), diana.getId()));
        
        System.out.println("Created group: " + tripGroup);
        
        // ==================== Add Expenses ====================
        System.out.println("\n===== ADDING EXPENSES =====\n");
        
        // Equal split expense
        Expense expense1 = service.addEqualExpense(
            tripGroup.getId(),
            alice.getId(),
            new BigDecimal("100.00"),
            "Dinner",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), diana.getId()));
        System.out.println("Added: " + expense1);
        
        // Another equal split
        Expense expense2 = service.addEqualExpense(
            tripGroup.getId(),
            bob.getId(),
            new BigDecimal("60.00"),
            "Groceries",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId()));
        System.out.println("Added: " + expense2);
        
        // Exact split expense
        List<Split> exactSplits = Arrays.asList(
            new ExactSplit(alice.getId(), new BigDecimal("50.00")),
            new ExactSplit(bob.getId(), new BigDecimal("30.00")),
            new ExactSplit(charlie.getId(), new BigDecimal("20.00")));
        
        Expense expense3 = service.addExpense(
            tripGroup.getId(),
            charlie.getId(),
            new BigDecimal("100.00"),
            "Gas",
            exactSplits);
        System.out.println("Added: " + expense3);
        
        // Percentage split expense
        List<Split> percentSplits = Arrays.asList(
            new PercentSplit(alice.getId(), new BigDecimal("40")),
            new PercentSplit(bob.getId(), new BigDecimal("35")),
            new PercentSplit(diana.getId(), new BigDecimal("25")));
        
        Expense expense4 = service.addExpense(
            tripGroup.getId(),
            diana.getId(),
            new BigDecimal("200.00"),
            "Hotel",
            percentSplits);
        System.out.println("Added: " + expense4);
        
        // ==================== Check Balances ====================
        System.out.println("\n===== CURRENT BALANCES =====");
        
        service.printBalances(alice.getId());
        service.printBalances(bob.getId());
        service.printBalances(charlie.getId());
        service.printBalances(diana.getId());
        
        // ==================== Simplify Debts ====================
        System.out.println("\n===== SIMPLIFIED SETTLEMENTS =====\n");
        
        List<Settlement> settlements = service.simplifyDebts(tripGroup.getId());
        
        System.out.println("Optimal settlements:");
        for (Settlement s : settlements) {
            User from = service.getUser(s.getFromUserId());
            User to = service.getUser(s.getToUserId());
            System.out.printf("  %s pays %s: $%.2f%n",
                from.getName(), to.getName(), s.getAmount());
        }
        
        // ==================== Settle Up ====================
        System.out.println("\n===== SETTLING UP =====\n");
        
        // Find who Alice owes
        Map<String, BigDecimal> aliceBalances = service.getBalances(alice.getId());
        for (Map.Entry<String, BigDecimal> entry : aliceBalances.entrySet()) {
            if (entry.getValue().compareTo(BigDecimal.ZERO) < 0) {
                User creditor = service.getUser(entry.getKey());
                BigDecimal debt = entry.getValue().abs();
                
                System.out.printf("Alice settles $%.2f with %s%n", debt, creditor.getName());
                service.settleUp(alice.getId(), entry.getKey(), debt);
            }
        }
        
        System.out.println("\nAlice's balances after settlement:");
        service.printBalances(alice.getId());
        
        // ==================== Expense History ====================
        System.out.println("\n===== EXPENSE HISTORY =====\n");
        
        System.out.println("Group expenses:");
        for (Expense e : service.getGroupExpenses(tripGroup.getId())) {
            User payer = service.getUser(e.getPaidBy());
            System.out.printf("  %s - $%.2f by %s (%s)%n",
                e.getDescription(), e.getAmount(), payer.getName(), e.getSplitType());
        }
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Understand the Problem

**What is Splitwise?**
- Track shared expenses among friends
- Split bills equally or by custom amounts
- Track who owes whom
- Simplify debts to minimize transactions

**Key Challenges:**
- **Multiple split types**: Equal, exact, percentage
- **Balance tracking**: Who owes whom how much
- **Debt simplification**: Minimize number of settlements
- **Rounding**: Handle cents properly

---

### Phase 2: Design the Split Model

```java
// Step 1: Split type enum
public enum SplitType {
    EQUAL,      // Split equally
    EXACT,      // Exact amounts
    PERCENTAGE  // Percentage-based
}

// Step 2: Abstract split class
public abstract class Split {
    protected final String userId;
    protected BigDecimal amount;
    
    public abstract SplitType getType();
}

// Step 3: Concrete split types
public class EqualSplit extends Split {
    // Amount calculated later
}

public class ExactSplit extends Split {
    public ExactSplit(String userId, BigDecimal amount) {
        super(userId);
        this.amount = amount;  // Amount known upfront
    }
}

public class PercentSplit extends Split {
    private final BigDecimal percentage;
    // Amount calculated from percentage
}
```

---

### Phase 3: Design the Expense Model

```java
// Step 4: Expense with split calculation
public class Expense {
    private final String paidBy;
    private final BigDecimal amount;
    private final List<Split> splits;
    
    public Expense(...) {
        // ...
        calculateSplitAmounts();
        validateSplits();
    }
    
    private void calculateSplitAmounts() {
        if (splitType == SplitType.EQUAL) {
            BigDecimal each = amount.divide(
                BigDecimal.valueOf(splits.size()), 2, RoundingMode.HALF_UP);
            
            // Handle rounding error
            BigDecimal total = each.multiply(BigDecimal.valueOf(splits.size()));
            BigDecimal diff = amount.subtract(total);
            
            // Give extra cent to first person
            splits.get(0).setAmount(each.add(diff));
            for (int i = 1; i < splits.size(); i++) {
                splits.get(i).setAmount(each);
            }
        }
    }
}
```

**Why handle rounding?**
```
$100 split 3 ways:
  $100 / 3 = $33.33 each
  $33.33 × 3 = $99.99 (missing $0.01!)

Solution: Give extra cent to first person
  Person 1: $33.34
  Person 2: $33.33
  Person 3: $33.33
  Total: $100.00 ✓
```

---

### Phase 4: Design the Balance Tracking

```java
// Step 5: User with balance tracking
public class User {
    // Balance with other users
    // Positive = they owe me
    // Negative = I owe them
    private final Map<String, BigDecimal> balances;
    
    public void updateBalance(String otherUserId, BigDecimal amount) {
        balances.merge(otherUserId, amount, BigDecimal::add);
        
        // Remove zero balances
        if (balances.get(otherUserId).compareTo(BigDecimal.ZERO) == 0) {
            balances.remove(otherUserId);
        }
    }
}
```

---

### Phase 5: Implement Debt Simplification

```java
// Step 6: Simplify debts algorithm
public List<Settlement> simplifyDebts(String groupId) {
    // Calculate net balance for each member
    Map<String, BigDecimal> netBalances = new HashMap<>();
    
    for (String memberId : group.getMemberIds()) {
        BigDecimal net = BigDecimal.ZERO;
        User user = getUser(memberId);
        
        for (String otherId : group.getMemberIds()) {
            net = net.add(user.getBalanceWith(otherId));
        }
        
        if (net.compareTo(BigDecimal.ZERO) != 0) {
            netBalances.put(memberId, net);
        }
    }
    
    // Separate into creditors and debtors
    PriorityQueue<Entry> creditors = new PriorityQueue<>(byAmountDesc);
    PriorityQueue<Entry> debtors = new PriorityQueue<>(byAmountAsc);
    
    // Match creditors with debtors
    while (!creditors.isEmpty() && !debtors.isEmpty()) {
        Entry creditor = creditors.poll();
        Entry debtor = debtors.poll();
        
        BigDecimal amount = min(creditor.amount, abs(debtor.amount));
        settlements.add(new Settlement(debtor.id, creditor.id, amount));
        
        // Put back remaining if any
        // ...
    }
    
    return settlements;
}
```

---

### Phase 6: Threading Model and Concurrency Control

**Threading Model:**

This system handles **concurrent expense additions**:
- Multiple users adding expenses simultaneously
- Balance updates must be atomic
- Debt simplification can run periodically or on-demand

**Concurrency Control:**

```java
public class ExpenseService {
    private final ConcurrentHashMap<String, User> users;
    private final ConcurrentHashMap<String, Expense> expenses;
    private final ReentrantLock balanceLock = new ReentrantLock();
    
    public Expense addExpense(...) {
        Expense expense = new Expense(...);
        
        balanceLock.lock();
        try {
            // Atomically update all balances
            for (Split split : expense.getSplits()) {
                User paidBy = users.get(expense.getPaidBy());
                User splitUser = users.get(split.getUserId());
                
                // Paid by gets positive balance
                paidBy.updateBalance(splitUser.getId(), split.getAmount());
                
                // Split user gets negative balance
                splitUser.updateBalance(paidBy.getId(), split.getAmount().negate());
            }
            
            expenses.put(expense.getId(), expense);
            return expense;
        } finally {
            balanceLock.unlock();
        }
    }
}
```

---
    
    protected final String userId;
    protected BigDecimal amount;
    
    public Split(String userId) {
        this.userId = userId;
    }
    
    public String getUserId() { return userId; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    
    public abstract SplitType getType();
}
```

```java
// EqualSplit.java
package com.splitwise;

public class EqualSplit extends Split {
    
    public EqualSplit(String userId) {
        super(userId);
    }
    
    @Override
    public SplitType getType() {
        return SplitType.EQUAL;
    }
}
```

```java
// ExactSplit.java
package com.splitwise;

import java.math.BigDecimal;

public class ExactSplit extends Split {
    
    public ExactSplit(String userId, BigDecimal amount) {
        super(userId);
        this.amount = amount;
    }
    
    @Override
    public SplitType getType() {
        return SplitType.EXACT;
    }
}
```

```java
// PercentSplit.java
package com.splitwise;

import java.math.BigDecimal;

public class PercentSplit extends Split {
    
    private final BigDecimal percentage;
    
    public PercentSplit(String userId, BigDecimal percentage) {
        super(userId);
        this.percentage = percentage;
    }
    
    public BigDecimal getPercentage() { return percentage; }
    
    @Override
    public SplitType getType() {
        return SplitType.PERCENTAGE;
    }
}
```

### 2.3 User Class

```java
// User.java
package com.splitwise;

import java.math.BigDecimal;
import java.util.*;

/**
 * Represents a user in the expense sharing system.
 */
public class User {
    
    private final String id;
    private final String name;
    private final String email;
    
    // Balance with other users: positive = they owe me, negative = I owe them
    private final Map<String, BigDecimal> balances;
    
    public User(String name, String email) {
        this.id = "USR-" + System.currentTimeMillis() % 100000;
        this.name = name;
        this.email = email;
        this.balances = new HashMap<>();
    }
    
    public void updateBalance(String otherUserId, BigDecimal amount) {
        balances.merge(otherUserId, amount, BigDecimal::add);
        
        // Remove zero balances
        if (balances.get(otherUserId).compareTo(BigDecimal.ZERO) == 0) {
            balances.remove(otherUserId);
        }
    }
    
    public BigDecimal getBalanceWith(String otherUserId) {
        return balances.getOrDefault(otherUserId, BigDecimal.ZERO);
    }
    
    public BigDecimal getTotalOwed() {
        return balances.values().stream()
            .filter(b -> b.compareTo(BigDecimal.ZERO) > 0)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    public BigDecimal getTotalOwing() {
        return balances.values().stream()
            .filter(b -> b.compareTo(BigDecimal.ZERO) < 0)
            .map(BigDecimal::abs)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    public Map<String, BigDecimal> getBalances() {
        return Collections.unmodifiableMap(balances);
    }
    
    @Override
    public String toString() {
        return String.format("%s (%s)", name, email);
    }
}
```

### 2.4 Expense Class

```java
// Expense.java
package com.splitwise;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Represents an expense in the system.
 */
public class Expense {
    
    private final String id;
    private final String description;
    private final BigDecimal amount;
    private final String paidBy;
    private final List<Split> splits;
    private final SplitType splitType;
    private final LocalDateTime timestamp;
    private final String groupId;
    
    public Expense(String description, BigDecimal amount, String paidBy,
                   List<Split> splits, String groupId) {
        this.id = "EXP-" + System.currentTimeMillis() % 100000;
        this.description = description;
        this.amount = amount;
        this.paidBy = paidBy;
        this.splits = new ArrayList<>(splits);
        this.splitType = splits.isEmpty() ? SplitType.EQUAL : splits.get(0).getType();
        this.timestamp = LocalDateTime.now();
        this.groupId = groupId;
        
        calculateSplitAmounts();
        validateSplits();
    }
    
    private void calculateSplitAmounts() {
        if (splitType == SplitType.EQUAL) {
            BigDecimal splitAmount = amount.divide(
                BigDecimal.valueOf(splits.size()), 2, RoundingMode.HALF_UP);
            
            // Handle rounding - give extra cent to first person
            BigDecimal total = splitAmount.multiply(BigDecimal.valueOf(splits.size()));
            BigDecimal diff = amount.subtract(total);
            
            for (int i = 0; i < splits.size(); i++) {
                if (i == 0) {
                    splits.get(i).setAmount(splitAmount.add(diff));
                } else {
                    splits.get(i).setAmount(splitAmount);
                }
            }
        } else if (splitType == SplitType.PERCENTAGE) {
            for (Split split : splits) {
                PercentSplit ps = (PercentSplit) split;
                BigDecimal splitAmount = amount.multiply(ps.getPercentage())
                    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
                split.setAmount(splitAmount);
            }
        }
        // EXACT splits already have amounts set
    }
    
    private void validateSplits() {
        BigDecimal totalSplit = splits.stream()
            .map(Split::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        if (totalSplit.compareTo(amount) != 0) {
            throw new IllegalArgumentException(
                "Split amounts don't add up to total: " + totalSplit + " vs " + amount);
        }
    }
    
    // Getters
    public String getId() { return id; }
    public String getDescription() { return description; }
    public BigDecimal getAmount() { return amount; }
    public String getPaidBy() { return paidBy; }
    public List<Split> getSplits() { return Collections.unmodifiableList(splits); }
    public SplitType getSplitType() { return splitType; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public String getGroupId() { return groupId; }
    
    @Override
    public String toString() {
        return String.format("%s: $%.2f paid by %s (%s split)",
            description, amount, paidBy, splitType);
    }
}
```

### 2.5 Group Class

```java
// Group.java
package com.splitwise;

import java.util.*;

/**
 * Represents a group for expense sharing.
 */
public class Group {
    
    private final String id;
    private final String name;
    private final Set<String> memberIds;
    private final List<String> expenseIds;
    
    public Group(String name) {
        this.id = "GRP-" + System.currentTimeMillis() % 100000;
        this.name = name;
        this.memberIds = new HashSet<>();
        this.expenseIds = new ArrayList<>();
    }
    
    public void addMember(String userId) {
        memberIds.add(userId);
    }
    
    public void removeMember(String userId) {
        memberIds.remove(userId);
    }
    
    public void addExpense(String expenseId) {
        expenseIds.add(0, expenseId);  // Most recent first
    }
    
    public boolean hasMember(String userId) {
        return memberIds.contains(userId);
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    
    public Set<String> getMemberIds() {
        return Collections.unmodifiableSet(memberIds);
    }
    
    public List<String> getExpenseIds() {
        return Collections.unmodifiableList(expenseIds);
    }
    
    @Override
    public String toString() {
        return String.format("%s (%d members)", name, memberIds.size());
    }
}
```

### 2.6 Settlement Class

```java
// Settlement.java
package com.splitwise;

import java.math.BigDecimal;

/**
 * Represents a settlement suggestion.
 */
public class Settlement {
    
    private final String fromUserId;
    private final String toUserId;
    private final BigDecimal amount;
    
    public Settlement(String fromUserId, String toUserId, BigDecimal amount) {
        this.fromUserId = fromUserId;
        this.toUserId = toUserId;
        this.amount = amount;
    }
    
    public String getFromUserId() { return fromUserId; }
    public String getToUserId() { return toUserId; }
    public BigDecimal getAmount() { return amount; }
    
    @Override
    public String toString() {
        return String.format("%s pays %s: $%.2f", fromUserId, toUserId, amount);
    }
}
```

### 2.7 ExpenseService Class

```java
// ExpenseService.java
package com.splitwise;

import java.math.BigDecimal;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Main service for expense sharing operations.
 */
public class ExpenseService {
    
    private final Map<String, User> users;
    private final Map<String, Group> groups;
    private final Map<String, Expense> expenses;
    
    public ExpenseService() {
        this.users = new HashMap<>();
        this.groups = new HashMap<>();
        this.expenses = new HashMap<>();
    }
    
    // ==================== User Management ====================
    
    public User createUser(String name, String email) {
        User user = new User(name, email);
        users.put(user.getId(), user);
        return user;
    }
    
    public User getUser(String userId) {
        User user = users.get(userId);
        if (user == null) {
            throw new IllegalArgumentException("User not found: " + userId);
        }
        return user;
    }
    
    // ==================== Group Management ====================
    
    public Group createGroup(String name, List<String> memberIds) {
        Group group = new Group(name);
        for (String memberId : memberIds) {
            getUser(memberId);  // Validate user exists
            group.addMember(memberId);
        }
        groups.put(group.getId(), group);
        return group;
    }
    
    public void addMemberToGroup(String groupId, String userId) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        getUser(userId);
        group.addMember(userId);
    }
    
    // ==================== Expense Management ====================
    
    public Expense addExpense(String groupId, String paidBy, BigDecimal amount,
                             String description, List<Split> splits) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        
        // Validate payer is in group
        if (!group.hasMember(paidBy)) {
            throw new IllegalArgumentException("Payer not in group");
        }
        
        // Validate all split users are in group
        for (Split split : splits) {
            if (!group.hasMember(split.getUserId())) {
                throw new IllegalArgumentException(
                    "Split user not in group: " + split.getUserId());
            }
        }
        
        Expense expense = new Expense(description, amount, paidBy, splits, groupId);
        expenses.put(expense.getId(), expense);
        group.addExpense(expense.getId());
        
        // Update balances
        updateBalances(expense);
        
        return expense;
    }
    
    public Expense addEqualExpense(String groupId, String paidBy, BigDecimal amount,
                                   String description, List<String> participantIds) {
        List<Split> splits = participantIds.stream()
            .map(EqualSplit::new)
            .collect(Collectors.toList());
        return addExpense(groupId, paidBy, amount, description, splits);
    }
    
    private void updateBalances(Expense expense) {
        String paidBy = expense.getPaidBy();
        User payer = getUser(paidBy);
        
        for (Split split : expense.getSplits()) {
            String owerId = split.getUserId();
            BigDecimal owedAmount = split.getAmount();
            
            if (!owerId.equals(paidBy)) {
                // Payer is owed this amount by ower
                payer.updateBalance(owerId, owedAmount);
                
                // Ower owes this amount to payer
                User ower = getUser(owerId);
                ower.updateBalance(paidBy, owedAmount.negate());
            }
        }
    }
    
    // ==================== Balance Queries ====================
    
    public Map<String, BigDecimal> getBalances(String userId) {
        return getUser(userId).getBalances();
    }
    
    public BigDecimal getBalance(String userId1, String userId2) {
        return getUser(userId1).getBalanceWith(userId2);
    }
    
    public void printBalances(String userId) {
        User user = getUser(userId);
        Map<String, BigDecimal> balances = user.getBalances();
        
        System.out.println("\nBalances for " + user.getName() + ":");
        
        if (balances.isEmpty()) {
            System.out.println("  All settled up!");
            return;
        }
        
        for (Map.Entry<String, BigDecimal> entry : balances.entrySet()) {
            User other = getUser(entry.getKey());
            BigDecimal amount = entry.getValue();
            
            if (amount.compareTo(BigDecimal.ZERO) > 0) {
                System.out.printf("  %s owes you $%.2f%n", other.getName(), amount);
            } else {
                System.out.printf("  You owe %s $%.2f%n", other.getName(), amount.abs());
            }
        }
    }
    
    // ==================== Settlement ====================
    
    public void settleUp(String fromUserId, String toUserId, BigDecimal amount) {
        User from = getUser(fromUserId);
        User to = getUser(toUserId);
        
        BigDecimal currentBalance = from.getBalanceWith(toUserId);
        
        if (currentBalance.compareTo(BigDecimal.ZERO) >= 0) {
            throw new IllegalStateException(
                from.getName() + " doesn't owe " + to.getName() + " anything");
        }
        
        if (amount.compareTo(currentBalance.abs()) > 0) {
            throw new IllegalArgumentException("Settlement amount exceeds debt");
        }
        
        from.updateBalance(toUserId, amount);
        to.updateBalance(fromUserId, amount.negate());
    }
    
    // ==================== Debt Simplification ====================
    
    /**
     * Simplifies debts within a group to minimize transactions.
     * Uses a greedy algorithm to match creditors with debtors.
     */
    public List<Settlement> simplifyDebts(String groupId) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        
        // Calculate net balance for each member
        Map<String, BigDecimal> netBalances = new HashMap<>();
        
        for (String memberId : group.getMemberIds()) {
            BigDecimal net = BigDecimal.ZERO;
            User user = getUser(memberId);
            
            for (String otherId : group.getMemberIds()) {
                if (!otherId.equals(memberId)) {
                    net = net.add(user.getBalanceWith(otherId));
                }
            }
            
            if (net.compareTo(BigDecimal.ZERO) != 0) {
                netBalances.put(memberId, net);
            }
        }
        
        // Separate into creditors (positive) and debtors (negative)
        PriorityQueue<Map.Entry<String, BigDecimal>> creditors = 
            new PriorityQueue<>((a, b) -> b.getValue().compareTo(a.getValue()));
        PriorityQueue<Map.Entry<String, BigDecimal>> debtors = 
            new PriorityQueue<>((a, b) -> a.getValue().compareTo(b.getValue()));
        
        for (Map.Entry<String, BigDecimal> entry : netBalances.entrySet()) {
            if (entry.getValue().compareTo(BigDecimal.ZERO) > 0) {
                creditors.offer(entry);
            } else if (entry.getValue().compareTo(BigDecimal.ZERO) < 0) {
                debtors.offer(entry);
            }
        }
        
        // Match creditors with debtors
        List<Settlement> settlements = new ArrayList<>();
        
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            Map.Entry<String, BigDecimal> creditor = creditors.poll();
            Map.Entry<String, BigDecimal> debtor = debtors.poll();
            
            BigDecimal credit = creditor.getValue();
            BigDecimal debt = debtor.getValue().abs();
            
            BigDecimal settlementAmount = credit.min(debt);
            
            settlements.add(new Settlement(
                debtor.getKey(), creditor.getKey(), settlementAmount));
            
            // Update remaining balances
            BigDecimal remainingCredit = credit.subtract(settlementAmount);
            BigDecimal remainingDebt = debt.subtract(settlementAmount);
            
            if (remainingCredit.compareTo(BigDecimal.ZERO) > 0) {
                creditors.offer(new AbstractMap.SimpleEntry<>(
                    creditor.getKey(), remainingCredit));
            }
            
            if (remainingDebt.compareTo(BigDecimal.ZERO) > 0) {
                debtors.offer(new AbstractMap.SimpleEntry<>(
                    debtor.getKey(), remainingDebt.negate()));
            }
        }
        
        return settlements;
    }
    
    // ==================== Expense History ====================
    
    public List<Expense> getGroupExpenses(String groupId) {
        Group group = groups.get(groupId);
        if (group == null) {
            throw new IllegalArgumentException("Group not found");
        }
        
        return group.getExpenseIds().stream()
            .map(expenses::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
    
    public List<Expense> getUserExpenses(String userId) {
        getUser(userId);
        
        return expenses.values().stream()
            .filter(e -> e.getPaidBy().equals(userId) ||
                        e.getSplits().stream()
                            .anyMatch(s -> s.getUserId().equals(userId)))
            .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
            .collect(Collectors.toList());
    }
}
```

### 2.8 Demo Application

```java
// SplitwiseDemo.java
package com.splitwise;

import java.math.BigDecimal;
import java.util.*;

public class SplitwiseDemo {
    
    public static void main(String[] args) {
        System.out.println("=== SPLITWISE EXPENSE SHARING DEMO ===\n");
        
        ExpenseService service = new ExpenseService();
        
        // ==================== Create Users ====================
        System.out.println("===== CREATING USERS =====\n");
        
        User alice = service.createUser("Alice", "alice@email.com");
        User bob = service.createUser("Bob", "bob@email.com");
        User charlie = service.createUser("Charlie", "charlie@email.com");
        User diana = service.createUser("Diana", "diana@email.com");
        
        System.out.println("Created: " + alice);
        System.out.println("Created: " + bob);
        System.out.println("Created: " + charlie);
        System.out.println("Created: " + diana);
        
        // ==================== Create Group ====================
        System.out.println("\n===== CREATING GROUP =====\n");
        
        Group tripGroup = service.createGroup("Beach Trip",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), diana.getId()));
        
        System.out.println("Created group: " + tripGroup);
        
        // ==================== Add Expenses ====================
        System.out.println("\n===== ADDING EXPENSES =====\n");
        
        // Equal split expense
        Expense expense1 = service.addEqualExpense(
            tripGroup.getId(),
            alice.getId(),
            new BigDecimal("100.00"),
            "Dinner",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), diana.getId()));
        System.out.println("Added: " + expense1);
        
        // Another equal split
        Expense expense2 = service.addEqualExpense(
            tripGroup.getId(),
            bob.getId(),
            new BigDecimal("60.00"),
            "Groceries",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId()));
        System.out.println("Added: " + expense2);
        
        // Exact split expense
        List<Split> exactSplits = Arrays.asList(
            new ExactSplit(alice.getId(), new BigDecimal("50.00")),
            new ExactSplit(bob.getId(), new BigDecimal("30.00")),
            new ExactSplit(charlie.getId(), new BigDecimal("20.00")));
        
        Expense expense3 = service.addExpense(
            tripGroup.getId(),
            charlie.getId(),
            new BigDecimal("100.00"),
            "Gas",
            exactSplits);
        System.out.println("Added: " + expense3);
        
        // Percentage split expense
        List<Split> percentSplits = Arrays.asList(
            new PercentSplit(alice.getId(), new BigDecimal("40")),
            new PercentSplit(bob.getId(), new BigDecimal("35")),
            new PercentSplit(diana.getId(), new BigDecimal("25")));
        
        Expense expense4 = service.addExpense(
            tripGroup.getId(),
            diana.getId(),
            new BigDecimal("200.00"),
            "Hotel",
            percentSplits);
        System.out.println("Added: " + expense4);
        
        // ==================== Check Balances ====================
        System.out.println("\n===== CURRENT BALANCES =====");
        
        service.printBalances(alice.getId());
        service.printBalances(bob.getId());
        service.printBalances(charlie.getId());
        service.printBalances(diana.getId());
        
        // ==================== Simplify Debts ====================
        System.out.println("\n===== SIMPLIFIED SETTLEMENTS =====\n");
        
        List<Settlement> settlements = service.simplifyDebts(tripGroup.getId());
        
        System.out.println("Optimal settlements:");
        for (Settlement s : settlements) {
            User from = service.getUser(s.getFromUserId());
            User to = service.getUser(s.getToUserId());
            System.out.printf("  %s pays %s: $%.2f%n",
                from.getName(), to.getName(), s.getAmount());
        }
        
        // ==================== Settle Up ====================
        System.out.println("\n===== SETTLING UP =====\n");
        
        // Find who Alice owes
        Map<String, BigDecimal> aliceBalances = service.getBalances(alice.getId());
        for (Map.Entry<String, BigDecimal> entry : aliceBalances.entrySet()) {
            if (entry.getValue().compareTo(BigDecimal.ZERO) < 0) {
                User creditor = service.getUser(entry.getKey());
                BigDecimal debt = entry.getValue().abs();
                
                System.out.printf("Alice settles $%.2f with %s%n", debt, creditor.getName());
                service.settleUp(alice.getId(), entry.getKey(), debt);
            }
        }
        
        System.out.println("\nAlice's balances after settlement:");
        service.printBalances(alice.getId());
        
        // ==================== Expense History ====================
        System.out.println("\n===== EXPENSE HISTORY =====\n");
        
        System.out.println("Group expenses:");
        for (Expense e : service.getGroupExpenses(tripGroup.getId())) {
            User payer = service.getUser(e.getPaidBy());
---

## File Structure

```
com/splitwise/
├── SplitType.java
├── Split.java
├── EqualSplit.java
├── ExactSplit.java
├── PercentSplit.java
├── User.java
├── Expense.java
├── Group.java
├── Settlement.java
├── ExpenseService.java
└── SplitwiseDemo.java
```

