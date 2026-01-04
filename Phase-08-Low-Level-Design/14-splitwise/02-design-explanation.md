# 💸 Splitwise (Expense Sharing) - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Expense Sharing System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `User` | Store user data and balances | User model changes |
| `Group` | Manage group membership | Group rules change |
| `Expense` | Store expense details | Expense model changes |
| `Split` | Define split amount | Split logic changes |
| `Settlement` | Represent a settlement | Settlement model changes |
| `ExpenseService` | Coordinate operations | Business logic changes |

**SRP in Action:**

```java
// User ONLY manages user data and balances
public class User {
    private final Map<String, BigDecimal> balances;
    
    public void updateBalance(String otherUserId, BigDecimal amount) { }
    public BigDecimal getBalanceWith(String otherUserId) { }
}

// Expense ONLY stores expense data
public class Expense {
    private final List<Split> splits;
    
    private void calculateSplitAmounts() { }
    private void validateSplits() { }
}

// ExpenseService coordinates operations
public class ExpenseService {
    public Expense addExpense(...) { }
    public List<Settlement> simplifyDebts(String groupId) { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Split Types:**

```java
// Base class
public abstract class Split {
    protected final String userId;
    protected BigDecimal amount;
    
    public abstract SplitType getType();
}

// Existing types
public class EqualSplit extends Split { }
public class ExactSplit extends Split { }
public class PercentSplit extends Split { }

// New types - no changes to existing code
public class ShareSplit extends Split {
    private final int shares;
    
    @Override
    public SplitType getType() {
        return SplitType.SHARE;
    }
}

public class AdjustmentSplit extends Split {
    private final BigDecimal adjustment;  // +/- from equal
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All split types work interchangeably:**

```java
public class Expense {
    private final List<Split> splits;  // Any Split subtype works
    
    private void calculateSplitAmounts() {
        if (splitType == SplitType.EQUAL) {
            // Calculate for EqualSplit
        } else if (splitType == SplitType.PERCENTAGE) {
            // Calculate for PercentSplit
        }
        // ExactSplit already has amounts
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Splittable {
    BigDecimal calculateShare(BigDecimal total, int participants);
}

public interface Validatable {
    boolean validate(BigDecimal total);
}

public class EqualSplit implements Splittable { }
public class PercentSplit implements Splittable, Validatable { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface UserRepository {
    void save(User user);
    User findById(String id);
}

public interface ExpenseRepository {
    void save(Expense expense);
    List<Expense> findByGroup(String groupId);
}

public interface BalanceCalculator {
    Map<String, BigDecimal> calculateNetBalances(String groupId);
}

public class ExpenseService {
    private final UserRepository userRepo;
    private final ExpenseRepository expenseRepo;
    private final BalanceCalculator balanceCalc;
}
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. User stores user data, Expense stores expense details, Split defines split amount, ExpenseService coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new split types) without modifying existing code. Inheritance with Split base class enables this. | N/A | - |
| **LSP** | PASS | All Split subclasses properly implement the Split contract. They are substitutable in expense calculations. | N/A | - |
| **ISP** | PASS | Split abstract class is focused. Subclasses only implement what they need. No unused methods forced on clients. | N/A | - |
| **DIP** | WEAK | ExpenseService depends on concrete classes. Could depend on UserRepository, ExpenseRepository, and BalanceCalculator interfaces. Mentioned in DIP section but not fully implemented. | Extract repository and calculator interfaces | More abstraction layers, but improves testability and data access flexibility |

---

## Design Patterns Used

### 1. Strategy Pattern

**Where:** Split calculation strategies

```java
public interface SplitStrategy {
    void calculateAmounts(List<Split> splits, BigDecimal total);
}

public class EqualSplitStrategy implements SplitStrategy {
    @Override
    public void calculateAmounts(List<Split> splits, BigDecimal total) {
        BigDecimal each = total.divide(BigDecimal.valueOf(splits.size()), 
            2, RoundingMode.HALF_UP);
        splits.forEach(s -> s.setAmount(each));
    }
}

public class PercentSplitStrategy implements SplitStrategy {
    @Override
    public void calculateAmounts(List<Split> splits, BigDecimal total) {
        for (Split split : splits) {
            PercentSplit ps = (PercentSplit) split;
            split.setAmount(total.multiply(ps.getPercentage())
                .divide(BigDecimal.valueOf(100)));
        }
    }
}
```

---

### 2. Factory Pattern (Potential)

**Where:** Creating splits

```java
public class SplitFactory {
    public static List<Split> createEqualSplits(List<String> userIds) {
        return userIds.stream()
            .map(EqualSplit::new)
            .collect(Collectors.toList());
    }
    
    public static List<Split> createPercentSplits(
            Map<String, BigDecimal> userPercentages) {
        return userPercentages.entrySet().stream()
            .map(e -> new PercentSplit(e.getKey(), e.getValue()))
            .collect(Collectors.toList());
    }
}
```

---

### 3. Observer Pattern (Potential)

**Where:** Notifications for new expenses

```java
public interface ExpenseObserver {
    void onExpenseAdded(Expense expense);
    void onSettlement(Settlement settlement);
}

public class NotificationService implements ExpenseObserver {
    @Override
    public void onExpenseAdded(Expense expense) {
        for (Split split : expense.getSplits()) {
            if (!split.getUserId().equals(expense.getPaidBy())) {
                notify(split.getUserId(), 
                    "You owe $" + split.getAmount() + " for " + expense.getDescription());
            }
        }
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Using a Single Split Class with Type Field

**What it is:** Instead of having separate `EqualSplit`, `ExactSplit`, and `PercentSplit` classes, use a single `Split` class with a `splitType` enum field and conditional logic.

**Why rejected:** This violates the Open/Closed Principle. Adding new split types would require modifying the existing `Split` class and adding more conditional logic. The current inheritance-based approach allows extending functionality without modifying existing code.

**What breaks:** 
- Violates OCP - must modify existing code for new split types
- Single class becomes bloated with multiple responsibilities
- Harder to test individual split strategies in isolation
- Type safety is reduced (no compile-time checking for split-specific fields)

### Alternative 2: Using a Database-First Approach with ORM

**What it is:** Design the system around database entities first, using an ORM like Hibernate, with all balance calculations happening at the database level using SQL queries.

**Why rejected:** This adds unnecessary complexity for an in-memory system. The current approach keeps the domain model simple and focused on business logic. Database operations would require transaction management, connection pooling, and complex SQL queries that are overkill for this use case.

**What breaks:**
- Over-engineering for an in-memory system
- Performance overhead from database round-trips
- Complex SQL queries for balance calculations
- Harder to test without database setup
- Loses the simplicity of in-memory data structures

---

## Debt Simplification Algorithm

### Greedy Approach

```java
public List<Settlement> simplifyDebts(String groupId) {
    // Step 1: Calculate net balance for each user
    Map<String, BigDecimal> netBalances = calculateNetBalances(groupId);
    
    // Step 2: Separate into creditors and debtors
    PriorityQueue<Entry> creditors = new PriorityQueue<>(byAmountDesc);
    PriorityQueue<Entry> debtors = new PriorityQueue<>(byAmountAsc);
    
    for (Entry e : netBalances.entrySet()) {
        if (e.getValue() > 0) creditors.add(e);
        else if (e.getValue() < 0) debtors.add(e);
    }
    
    // Step 3: Match creditors with debtors
    List<Settlement> settlements = new ArrayList<>();
    
    while (!creditors.isEmpty() && !debtors.isEmpty()) {
        Entry creditor = creditors.poll();
        Entry debtor = debtors.poll();
        
        BigDecimal amount = min(creditor.amount, abs(debtor.amount));
        settlements.add(new Settlement(debtor.id, creditor.id, amount));
        
        // Put back remaining balance
        if (creditor.amount > amount) {
            creditors.add(new Entry(creditor.id, creditor.amount - amount));
        }
        if (abs(debtor.amount) > amount) {
            debtors.add(new Entry(debtor.id, debtor.amount + amount));
        }
    }
    
    return settlements;
}
```

**Time Complexity:** O(N log N) where N = number of users
**Space Complexity:** O(N)

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle expense deletion?

**Answer:**

```java
public void deleteExpense(String expenseId) {
    Expense expense = expenses.get(expenseId);
    if (expense == null) return;

    // Reverse the balance updates
    String paidBy = expense.getPaidBy();
    User payer = getUser(paidBy);

    for (Split split : expense.getSplits()) {
        if (!split.getUserId().equals(paidBy)) {
            // Reverse: payer no longer owed
            payer.updateBalance(split.getUserId(), split.getAmount().negate());

            // Reverse: ower no longer owes
            User ower = getUser(split.getUserId());
            ower.updateBalance(paidBy, split.getAmount());
        }
    }

    expenses.remove(expenseId);
    groups.get(expense.getGroupId()).removeExpense(expenseId);
}
```

---

### Q2: How would you handle partial payments?

**Answer:**

```java
public class PartialPayment {
    private final String expenseId;
    private final String userId;
    private final BigDecimal amountPaid;
    private final LocalDateTime paidAt;
}

public class Split {
    private BigDecimal amountOwed;
    private BigDecimal amountPaid;

    public BigDecimal getRemainingAmount() {
        return amountOwed.subtract(amountPaid);
    }

    public void recordPayment(BigDecimal amount) {
        amountPaid = amountPaid.add(amount);
    }
}
```

---

### Q3: How would you implement expense comments?

**Answer:**

```java
public class ExpenseComment {
    private final String id;
    private final String expenseId;
    private final String authorId;
    private final String text;
    private final LocalDateTime timestamp;
}

public class Expense {
    private final List<ExpenseComment> comments;

    public void addComment(String authorId, String text) {
        comments.add(new ExpenseComment(this.id, authorId, text));
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add expense attachments** - Receipt photos
2. **Add activity feed** - Recent changes
3. **Add expense reminders** - Notify for settlements
4. **Add export functionality** - CSV/PDF reports
5. **Add budget tracking** - Set spending limits per category
6. **Add offline support** - Sync when online
7. **Add multi-currency support** - Handle different currencies
8. **Add recurring expenses** - Automate regular expenses

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `addExpense` | O(P) | P = participants |
| `getBalances` | O(1) | Direct map access |
| `simplifyDebts` | O(N log N) | N = group members |
| `settleUp` | O(1) | Update two balances |
| `getGroupExpenses` | O(E) | E = expenses |

### Space Complexity

| Component | Space |
|-----------|-------|
| Users | O(U) |
| Groups | O(G) |
| Expenses | O(E) |
| Balances per user | O(U) | worst case |

### Bottlenecks at Scale

**10x Usage (1K → 10K users, 100 → 1K expenses/group):**
- Problem: Debt simplification algorithm becomes slow (O(U²) worst case), balance calculation overhead grows, expense history storage increases
- Solution: Optimize simplification algorithm (greedy matching), cache computed balances, implement expense archiving
- Tradeoff: Algorithm complexity increases, cache invalidation logic needed

**100x Usage (1K → 100K users, 100 → 10K expenses/group):**
- Problem: Single instance can't handle large groups, simplification algorithm too slow for large user sets, balance storage exceeds memory
- Solution: Shard groups by group ID, use distributed computation for simplification, implement database-backed balance storage
- Tradeoff: Distributed system complexity, need cross-shard balance reconciliation


### Q1: How would you handle multi-currency expenses?

```java
public class CurrencyExpense extends Expense {
    private final Currency currency;
    private final BigDecimal exchangeRate;
    
    public BigDecimal getAmountInBaseCurrency() {
        return amount.multiply(exchangeRate);
    }
}

public class CurrencyService {
    public BigDecimal convert(BigDecimal amount, Currency from, Currency to) {
        BigDecimal rate = getExchangeRate(from, to);
        return amount.multiply(rate);
    }
}
```

### Q2: How would you implement recurring expenses?

```java
public class RecurringExpense {
    private final Expense template;
    private final RecurrencePattern pattern;
    private final LocalDate startDate;
    private final LocalDate endDate;
    
    public List<Expense> generateExpenses() {
        List<Expense> expenses = new ArrayList<>();
        LocalDate current = startDate;
        
        while (!current.isAfter(endDate)) {
            expenses.add(createExpenseFromTemplate(current));
            current = pattern.nextOccurrence(current);
        }
        
        return expenses;
    }
}

public enum RecurrencePattern {
    DAILY, WEEKLY, MONTHLY, YEARLY
}
```

### Q3: How would you handle expense categories?

```java
public enum ExpenseCategory {
    FOOD, TRANSPORT, ACCOMMODATION, ENTERTAINMENT, UTILITIES, OTHER
}

public class Expense {
    private ExpenseCategory category;
}

public class ExpenseService {
    public Map<ExpenseCategory, BigDecimal> getCategoryBreakdown(String groupId) {
        return getGroupExpenses(groupId).stream()
            .collect(Collectors.groupingBy(
                Expense::getCategory,
                Collectors.reducing(BigDecimal.ZERO, 
                    Expense::getAmount, BigDecimal::add)));
    }
}
```

### Q4: How would you implement expense receipts/images?

```java
public class ExpenseReceipt {
    private final String expenseId;
    private final String imageUrl;
    private final LocalDateTime uploadedAt;
}

public class Expense {
    private List<ExpenseReceipt> receipts;
    
    public void addReceipt(String imageUrl) {
        receipts.add(new ExpenseReceipt(this.id, imageUrl, LocalDateTime.now()));
    }
}
```

