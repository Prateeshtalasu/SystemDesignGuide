# 💸 Splitwise (Expense Sharing) - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Group Expense Split

```
Group: "Trip", Members: Alice, Bob, Carol

Expense 1: Alice pays $90 dinner, split equally
- Each owes: $30
- Bob owes Alice: $30
- Carol owes Alice: $30

Expense 2: Bob pays $60 taxi, split equally
- Each owes: $20
- Alice owes Bob: $20
- Carol owes Bob: $20

Net Balances:
- Alice: +$90 - $20 = +$70 (is owed)
- Bob: -$30 + $60 - $20 = +$10 (is owed)
- Carol: -$30 - $20 = -$50 (owes)

Simplified: Carol pays Alice $50, Carol pays Bob $10
```

**Final State:**

```
Net balances correctly calculated
Debt simplification reduces transactions from 8 to 3
All operations completed successfully
```

---

### Scenario 2: Failure/Invalid Input - Percentage Split Not 100%

**Initial State:**

```
Group: "Trip", Members: Alice, Bob, Carol
Expense: $100 dinner
Split: Alice 40%, Bob 40%, Carol 30% (total 110%, invalid)
```

**Step-by-step:**

1. `expenseService.addExpense("Alice", $100, "Trip", percentSplits)`

   - Percent splits: Alice 40%, Bob 40%, Carol 30%
   - Validate: Total = 40 + 40 + 30 = 110% ≠ 100%
   - Validation fails
   - Throws IllegalArgumentException("Total percentage must equal 100%")
   - No expense created

2. `expenseService.addExpense("Alice", $0, "Trip", equalSplits)` (invalid input)

   - Zero amount → throws IllegalArgumentException("Amount must be positive")
   - No state change

3. `expenseService.addExpense(null, $100, "Trip", splits)` (invalid input)
   - Null paidBy → throws IllegalArgumentException
   - No state change

**Final State:**

```
Group: No new expense added
Balances unchanged
Invalid inputs properly rejected
```

---

### Scenario 3: Concurrency/Race Condition - Concurrent Expense Additions

**Initial State:**

```
Group: "Trip", Members: Alice, Bob, Carol
Thread A: Alice adds expense $100 (equal split)
Thread B: Bob adds expense $60 (equal split, concurrent)
Both update user balances concurrently
```

**Step-by-step (simulating concurrent expense additions):**

**Thread A:** `expenseService.addExpense("Alice", $100, "Trip", equalSplits)` at time T0
**Thread B:** `expenseService.addExpense("Bob", $60, "Trip", equalSplits)` at time T0 (concurrent)

1. **Thread A:** Creates Expense object

   - Calculates splits: Each owes $33.33 (Alice, Bob, Carol)
   - Validates splits → passes
   - Calls `alice.updateBalance("Bob", -$33.33)` (Alice paid, Bob owes)
   - Acquires lock on Alice's balance map
   - Updates balance: Bob balance = -$33.33
   - Releases lock
   - Updates Bob and Carol balances similarly
   - Adds expense to group

2. **Thread B:** Creates Expense object (concurrent)

   - Calculates splits: Each owes $20 (Alice, Bob, Carol)
   - Validates splits → passes
   - Calls `bob.updateBalance("Alice", -$20)` (Bob paid, Alice owes)
   - Acquires lock on Bob's balance map
   - Updates balance: Alice balance = -$20
   - Releases lock
   - Updates Alice and Carol balances
   - Adds expense to group

3. **Both threads operate on different user balance maps:**
   - Thread A updates: Alice→Bob, Alice→Carol, Bob→Alice, Carol→Alice
   - Thread B updates: Bob→Alice, Bob→Carol, Alice→Bob, Carol→Bob
   - Each user's balance map is synchronized independently
   - No conflicts as each thread updates different user objects

**Final State:**

```
Expense 1: $100 (Alice paid) - added successfully
Expense 2: $60 (Bob paid) - added successfully
Balances correctly updated for both expenses
No race conditions, proper per-user synchronization
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions

- **Percentage Split ≠ 100%**: Reject
- **Zero Amount**: Reject
- **Single Person Group**: No split needed
- **Self-Expense**: Valid (paid for self)

---

## Visual Trace: Debt Simplification

```
Group: Alice, Bob, Charlie, Diana

Expenses:
1. Alice paid $100 dinner (equal split 4 ways)
2. Bob paid $60 groceries (equal split with Alice, Charlie)
3. Charlie paid $100 gas (Alice $50, Bob $30, Charlie $20)
4. Diana paid $200 hotel (Alice 40%, Bob 35%, Diana 25%)

Step 1: Calculate individual debts
┌─────────────────────────────────────────────────────────────────┐
│ From Expense 1 (Dinner $100, 4-way):                            │
│   Alice paid, each owes $25                                     │
│   Bob → Alice: $25                                              │
│   Charlie → Alice: $25                                          │
│   Diana → Alice: $25                                            │
├─────────────────────────────────────────────────────────────────┤
│ From Expense 2 (Groceries $60, 3-way):                          │
│   Bob paid, each owes $20                                       │
│   Alice → Bob: $20                                              │
│   Charlie → Bob: $20                                            │
├─────────────────────────────────────────────────────────────────┤
│ From Expense 3 (Gas $100):                                      │
│   Charlie paid                                                  │
│   Alice → Charlie: $50                                          │
│   Bob → Charlie: $30                                            │
├─────────────────────────────────────────────────────────────────┤
│ From Expense 4 (Hotel $200):                                    │
│   Diana paid                                                    │
│   Alice → Diana: $80 (40%)                                      │
│   Bob → Diana: $70 (35%)                                        │
└─────────────────────────────────────────────────────────────────┘

Step 2: Calculate net balances
┌─────────────────────────────────────────────────────────────────┐
│ Alice:                                                          │
│   Received: $25 + $25 + $25 = $75                              │
│   Owes: $20 + $50 + $80 = $150                                 │
│   Net: -$75 (owes $75)                                         │
├─────────────────────────────────────────────────────────────────┤
│ Bob:                                                            │
│   Received: $20 + $20 = $40                                    │
│   Owes: $25 + $30 + $70 = $125                                 │
│   Net: -$85 (owes $85)                                         │
├─────────────────────────────────────────────────────────────────┤
│ Charlie:                                                        │
│   Received: $50 + $30 = $80                                    │
│   Owes: $25 + $20 = $45                                        │
│   Net: +$35 (owed $35)                                         │
├─────────────────────────────────────────────────────────────────┤
│ Diana:                                                          │
│   Received: $80 + $70 = $150                                   │
│   Owes: $25 = $25                                              │
│   Net: +$125 (owed $125)                                       │
└─────────────────────────────────────────────────────────────────┘

Step 3: Match creditors with debtors
┌─────────────────────────────────────────────────────────────────┐
│ Creditors (owed): Diana (+$125), Charlie (+$35)                │
│ Debtors (owe): Bob (-$85), Alice (-$75)                        │
│                                                                 │
│ Match 1: Bob pays Diana $85                                    │
│   Diana remaining: $125 - $85 = $40                            │
│                                                                 │
│ Match 2: Alice pays Diana $40                                  │
│   Diana remaining: $0                                          │
│   Alice remaining: $75 - $40 = $35                             │
│                                                                 │
│ Match 3: Alice pays Charlie $35                                │
│   Both settled                                                 │
└─────────────────────────────────────────────────────────────────┘

Result: 3 transactions instead of 8!
  Bob → Diana: $85
  Alice → Diana: $40
  Alice → Charlie: $35
```

---

## Testing Approach

### Unit Tests

```java
// SplitTest.java
public class SplitTest {

    @Test
    void testEqualSplit() {
        List<Split> splits = Arrays.asList(
            new EqualSplit("user1"),
            new EqualSplit("user2"),
            new EqualSplit("user3"));

        Expense expense = new Expense("Test", new BigDecimal("100.00"),
            "user1", splits, "group1");

        // First person gets extra cent from rounding
        assertEquals(new BigDecimal("33.34"), splits.get(0).getAmount());
        assertEquals(new BigDecimal("33.33"), splits.get(1).getAmount());
        assertEquals(new BigDecimal("33.33"), splits.get(2).getAmount());
    }

    @Test
    void testPercentSplit() {
        List<Split> splits = Arrays.asList(
            new PercentSplit("user1", new BigDecimal("50")),
            new PercentSplit("user2", new BigDecimal("30")),
            new PercentSplit("user3", new BigDecimal("20")));

        Expense expense = new Expense("Test", new BigDecimal("100.00"),
            "user1", splits, "group1");

        assertEquals(new BigDecimal("50.00"), splits.get(0).getAmount());
        assertEquals(new BigDecimal("30.00"), splits.get(1).getAmount());
        assertEquals(new BigDecimal("20.00"), splits.get(2).getAmount());
    }

    @Test
    void testInvalidSplitTotal() {
        List<Split> splits = Arrays.asList(
            new ExactSplit("user1", new BigDecimal("60.00")),
            new ExactSplit("user2", new BigDecimal("30.00")));  // Total: $90

        assertThrows(IllegalArgumentException.class, () ->
            new Expense("Test", new BigDecimal("100.00"), "user1", splits, "group1"));
    }
}
```

### Integration Tests

```java
// ExpenseServiceTest.java
public class ExpenseServiceTest {

    private ExpenseService service;
    private User alice, bob, charlie;
    private Group group;

    @BeforeEach
    void setUp() {
        service = new ExpenseService();
        alice = service.createUser("Alice", "alice@test.com");
        bob = service.createUser("Bob", "bob@test.com");
        charlie = service.createUser("Charlie", "charlie@test.com");
        group = service.createGroup("Test",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId()));
    }

    @Test
    void testBalanceAfterExpense() {
        service.addEqualExpense(group.getId(), alice.getId(),
            new BigDecimal("90.00"), "Dinner",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId()));

        // Bob and Charlie each owe Alice $30
        assertEquals(new BigDecimal("30.00"),
            service.getBalance(alice.getId(), bob.getId()));
        assertEquals(new BigDecimal("-30.00"),
            service.getBalance(bob.getId(), alice.getId()));
    }

    @Test
    void testDebtSimplification() {
        // Alice pays $60 (split 3 ways) → Bob, Charlie owe Alice $20 each
        service.addEqualExpense(group.getId(), alice.getId(),
            new BigDecimal("60.00"), "Expense 1",
            Arrays.asList(alice.getId(), bob.getId(), charlie.getId()));

        // Bob pays $30 (split with Alice) → Alice owes Bob $15
        service.addEqualExpense(group.getId(), bob.getId(),
            new BigDecimal("30.00"), "Expense 2",
            Arrays.asList(alice.getId(), bob.getId()));

        List<Settlement> settlements = service.simplifyDebts(group.getId());

        // Net: Alice owed $5 by Bob, $20 by Charlie
        // Should be 2 settlements
        assertEquals(2, settlements.size());
    }
}
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.
