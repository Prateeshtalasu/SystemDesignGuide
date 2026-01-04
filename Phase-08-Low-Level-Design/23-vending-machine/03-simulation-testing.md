# 🥤 Vending Machine - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Successful Purchase

**Initial State:**

```
VendingMachine: IDLE state
Balance: $0.00
Slot A1: Coca-Cola, $1.50, quantity: 5
```

**Step-by-step:**

1. `insertCoin(DOLLAR)`

   - State: IDLE → HAS_MONEY
   - Balance: $0.00 → $1.00

2. `insertCoin(QUARTER)`

   - State: HAS_MONEY (stays)
   - Balance: $1.00 → $1.25

3. `insertCoin(QUARTER)`

   - State: HAS_MONEY (stays)
   - Balance: $1.25 → $1.50

4. `selectProduct("A1")`

   - Product found: Coca-Cola @ $1.50
   - Balance ($1.50) >= Price ($1.50) ✓
   - State: HAS_MONEY → DISPENSING
   - Selected product: Coca-Cola

5. `dispense()`
   - Dispense Coca-Cola from slot A1
   - Change: $1.50 - $1.50 = $0.00 (no change)
   - State: DISPENSING → IDLE
   - Balance: $1.50 → $0.00
   - Slot A1 quantity: 5 → 4

**Final State:**

```
VendingMachine: IDLE state
Balance: $0.00
Slot A1: Coca-Cola, $1.50, quantity: 4
Product dispensed to user
```

---

### Scenario 2: Failure/Invalid Input - Insufficient Funds

**Initial State:**

```
VendingMachine: IDLE state
Balance: $0.00
Slot A1: Coca-Cola, $1.50, quantity: 5
```

**Step-by-step:**

1. `insertCoin(QUARTER)`

   - State: IDLE → HAS_MONEY
   - Balance: $0.00 → $0.25

2. `selectProduct("A1")`
   - Product found: Coca-Cola @ $1.50
   - Balance ($0.25) < Price ($1.50) ✗
   - Error: "Insufficient funds. Please insert $1.25 more"
   - State: HAS_MONEY (stays)
   - Balance: $0.25 (unchanged)

**Final State:**

```
VendingMachine: HAS_MONEY state
Balance: $0.25
Slot A1: Coca-Cola, $1.50, quantity: 5 (unchanged)
No product dispensed
```

---

### Scenario 3: Edge Case - Change Calculation with Overpayment

**Initial State:**

```
VendingMachine: IDLE state
Balance: $0.00
Slot A1: Coca-Cola, $1.25, quantity: 5
```

**Step-by-step:**

1. `insertCoin(DOLLAR)`

   - State: IDLE → HAS_MONEY
   - Balance: $0.00 → $1.00

2. `insertCoin(DOLLAR)`

   - State: HAS_MONEY (stays)
   - Balance: $1.00 → $2.00

3. `selectProduct("A1")`

   - Product found: Coca-Cola @ $1.25
   - Balance ($2.00) >= Price ($1.25) ✓
   - State: HAS_MONEY → DISPENSING
   - Selected product: Coca-Cola

4. `dispense()`
   - Dispense Coca-Cola from slot A1
   - Change needed: $2.00 - $1.25 = $0.75
   - Calculate change: [QUARTER, QUARTER, QUARTER] (75 cents)
   - State: DISPENSING → IDLE
   - Balance: $2.00 → $0.00
   - Slot A1 quantity: 5 → 4
   - Return change to user

**Final State:**

```
VendingMachine: IDLE state
Balance: $0.00
Slot A1: Coca-Cola, $1.25, quantity: 4
Product dispensed
Change returned: 3 quarters ($0.75)
```

---

### Scenario 4: Concurrency / Race Condition - Last Item Purchase Race

**Initial State:**

```
VendingMachine: IDLE state
Balance: $0.00
Slot A1: Coca-Cola, $1.50, quantity: 1 (last item)
```

**Concurrent Operations (Theoretical):**

**User 1 Timeline:**

1. `insertCoin(DOLLAR)` → Balance: $1.00, State: HAS_MONEY
2. `insertCoin(QUARTER)` → Balance: $1.25, State: HAS_MONEY
3. `insertCoin(QUARTER)` → Balance: $1.50, State: HAS_MONEY
4. `selectProduct("A1")` → Validates availability (quantity = 1 ✓)

**User 2 Timeline (concurrent):**

1. `insertCoin(DOLLAR)` → Balance: $1.00, State: HAS_MONEY (User 2's balance)
2. `insertCoin(QUARTER)` → Balance: $1.25, State: HAS_MONEY
3. `insertCoin(QUARTER)` → Balance: $1.50, State: HAS_MONEY
4. `selectProduct("A1")` → Validates availability (quantity = 1 ✓)

**Race Condition:**

- Both users pass availability check (quantity = 1)
- User 1's `dispense()` executes first → quantity: 1 → 0
- User 2's `dispense()` executes → quantity: 0 → fails to dispense

**Expected Behavior (with proper synchronization):**

- User 1 successfully dispenses the last item
- User 2 receives "Product sold out" error after validation but before dispense
- User 2's balance is refunded automatically

**Note:** Current implementation is single-threaded (per requirements), but in a concurrent scenario, this would require locking around the check-and-dispense operation to prevent the race condition.

---

### Visual Trace: Change Calculation

```
User inserts $2.00 for $1.25 product

Change needed: $2.00 - $1.25 = $0.75

┌─────────────────────────────────────────────────────────────────┐
│ calculateChange(75 cents)                                       │
├─────────────────────────────────────────────────────────────────┤
│ Try DOLLAR (100): 75 < 100, skip                               │
│ Try QUARTER (25): 75 >= 25                                      │
│   Add QUARTER, remaining: 50                                    │
│   50 >= 25, Add QUARTER, remaining: 25                         │
│   25 >= 25, Add QUARTER, remaining: 0                          │
│ Done!                                                           │
├─────────────────────────────────────────────────────────────────┤
│ Result: [QUARTER, QUARTER, QUARTER]                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## STEP 6: Edge Cases & Testing Strategy

### Edge Cases

| Category  | Edge Case                    | Expected Behavior                |
| --------- | ---------------------------- | -------------------------------- |
| Insert    | Insert coin in IDLE          | Transition to HAS_MONEY          |
| Insert    | Insert coin while dispensing | Reject, show message             |
| Select    | Select without money         | Show "insert money first"        |
| Select    | Select invalid code          | Show "invalid product"           |
| Select    | Select sold out item         | Show "sold out"                  |
| Select    | Insufficient funds           | Show required amount             |
| Dispense  | Exact change                 | No change returned               |
| Dispense  | Overpayment                  | Return correct change            |
| Refund    | Refund in IDLE               | Show "no money to refund"        |
| Refund    | Refund in HAS_MONEY          | Return all coins, go to IDLE     |
| Refund    | Refund while dispensing      | Reject                           |
| Change    | Cannot make exact change     | Handle gracefully                |
| Inventory | Restock sold out             | Transition from SOLD_OUT to IDLE |
| Inventory | Restock beyond capacity      | Cap at capacity                  |

### Unit Tests

```java
// VendingMachineTest.java
public class VendingMachineTest {

    private VendingMachine machine;

    @BeforeEach
    void setUp() {
        machine = new VendingMachine();
        machine.stockProduct("A1", new Product("A1", "Cola", 150), 5);
    }

    @Test
    void testSuccessfulPurchase() {
        machine.insertCoin(Coin.DOLLAR);
        machine.insertCoin(Coin.QUARTER);
        machine.insertCoin(Coin.QUARTER);
        machine.selectProduct("A1");

        assertEquals("IDLE", machine.getCurrentStateName());
        assertEquals(0, machine.getCurrentBalance());
        assertEquals(1, machine.getSalesHistory().size());
    }

    @Test
    void testInsufficientFunds() {
        machine.insertCoin(Coin.QUARTER);
        machine.selectProduct("A1");  // Needs $1.50

        assertEquals("HAS_MONEY", machine.getCurrentStateName());
        assertEquals(25, machine.getCurrentBalance());
    }

    @Test
    void testRefund() {
        machine.insertCoin(Coin.DOLLAR);
        machine.insertCoin(Coin.QUARTER);
        machine.refund();

        assertEquals("IDLE", machine.getCurrentStateName());
        assertEquals(0, machine.getCurrentBalance());
    }

    @Test
    void testSoldOut() {
        // Buy all items
        for (int i = 0; i < 5; i++) {
            machine.insertCoin(Coin.DOLLAR);
            machine.insertCoin(Coin.QUARTER);
            machine.insertCoin(Coin.QUARTER);
            machine.selectProduct("A1");
        }

        // Try to buy when sold out
        machine.insertCoin(Coin.DOLLAR);
        machine.insertCoin(Coin.QUARTER);
        machine.insertCoin(Coin.QUARTER);
        machine.selectProduct("A1");

        // Should still have money, product not dispensed
        assertEquals(150, machine.getCurrentBalance());
    }
}
```

```java
// ChangeCalculationTest.java
public class ChangeCalculationTest {

    @Test
    void testExactChange() {
        List<Coin> change = calculateChange(0);
        assertTrue(change.isEmpty());
    }

    @Test
    void testQuartersOnly() {
        List<Coin> change = calculateChange(75);
        assertEquals(3, change.size());
        assertTrue(change.stream().allMatch(c -> c == Coin.QUARTER));
    }

    @Test
    void testMixedCoins() {
        List<Coin> change = calculateChange(41);
        // 25 + 10 + 5 + 1 = 41
        assertEquals(4, change.size());
    }
}
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.
