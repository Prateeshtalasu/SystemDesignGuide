# 📚 Library Management System - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Complete Lending Cycle

```
Initial State:
- Book: "Effective Java" (ISBN: 978-0-13-468599-1)
- Copies: EJ-001 (AVAILABLE), EJ-002 (AVAILABLE), EJ-003 (AVAILABLE)
- Members: Alice (0 loans), Bob (0 loans)

Step 1: Alice checks out EJ-001
- Validate: alice.canBorrow() → true (active, no fines, 0 < 5 loans)
- Find item: catalog.findByBarcode("EJ-001") → BookItem
- Check: bookItem.isAvailable() → true
- Checkout: bookItem.checkout(alice, 14)
  - status = BORROWED
  - borrowedBy = alice
  - dueDate = today + 14 days
- Create Lending record
- alice.addLoan(lending)
- Send notification

State after Step 1:
- EJ-001: BORROWED by Alice, due in 14 days
- Alice: 1 loan

Step 2: Bob checks out EJ-002 and EJ-003
- Similar process
- EJ-002: BORROWED by Bob
- EJ-003: BORROWED by Bob
- Bob: 2 loans

Step 3: Alice tries to checkout EJ-003 (already borrowed)
- bookItem.isAvailable() → false (status = BORROWED)
- Return null, checkout fails

Step 4: Alice reserves "Effective Java"
- book.hasAvailableCopy() → false (all borrowed)
- Create Reservation(alice, book)
- Add to reservationsByBook queue
- alice.addReservation(reservation)

Step 5: Bob returns EJ-002
- lending.processReturn()
  - returnDate = today
  - Not overdue → no fine
- bookItem.returnItem()
  - status = AVAILABLE
- bob.removeLoan(lending)
- reservationService.processBookReturn(book)
  - Find Alice's reservation
  - reservation.fulfill()
  - Send notification to Alice

Final State:
- EJ-001: BORROWED by Alice
- EJ-002: AVAILABLE (reserved for Alice)
- EJ-003: BORROWED by Bob
- Alice: 1 loan, 1 fulfilled reservation
- Bob: 1 loan
```

---

### Scenario 2: Failure/Invalid Input - Member at Loan Limit

**Initial State:**

```
Member: Alice (4 active loans: BOOK-001, BOOK-002, BOOK-003, BOOK-004)
Library: MAX_BOOKS_ALLOWED = 5
Book: "New Book" (ISBN: 978-123-456-7890)
BookItem: NB-001 (AVAILABLE)
```

**Step-by-step:**

1. `lendingService.checkoutBook("ALICE-ID", "NB-001", 14)`

   - Validate: `alice.canBorrow()` → Check `activeLoans.size()`
   - Current loans: 4 < 5 (MAX_BOOKS_ALLOWED) → true ✓
   - Checkout succeeds
   - Alice: 5 active loans

2. `lendingService.checkoutBook("ALICE-ID", "ANOTHER-001", 14)` (attempts 6th book)
   - Validate: `alice.canBorrow()` → Check `activeLoans.size()`
   - Current loans: 5 >= 5 (MAX_BOOKS_ALLOWED) → false ✗
   - `canBorrow()` returns false
   - Checkout fails with error: "Member has reached maximum loan limit (5 books)"
   - BookItem remains AVAILABLE
   - No lending record created

**Final State:**

```
Alice: 5 active loans (at limit)
NB-001: BORROWED by Alice
ANOTHER-001: AVAILABLE (checkout rejected)
Error message returned to caller
```

---

### Scenario 3: Concurrency/Race Condition - Double Checkout

**Initial State:**

```
BookItem: "Popular Book" - PB-001 (AVAILABLE, status = AVAILABLE)
Librarian A: Thread 1
Librarian B: Thread 2
Member: Alice (canBorrow = true)
Member: Bob (canBorrow = true)
```

**Step-by-step (simulating concurrent checkout attempts):**

**Thread 1 (Librarian A):** `lendingService.checkoutBook("ALICE-ID", "PB-001", 14)` at time T0
**Thread 2 (Librarian B):** `lendingService.checkoutBook("BOB-ID", "PB-001", 14)` at time T0 (simultaneous)

1. **Thread 1:** Enters `BookItem.checkout(alice, 14)`

   - `synchronized (this)` lock acquired on PB-001
   - Check: `status == AVAILABLE` → true ✓
   - Status change: AVAILABLE → BORROWED (in progress)
   - Set `borrowedBy = alice`
   - Set `dueDate = today + 14 days`

2. **Thread 2:** Attempts to enter `BookItem.checkout(bob, 14)` (concurrent)

   - `synchronized (this)` lock WAIT (Thread 1 holds lock)
   - Thread 2 blocks, waiting for lock

3. **Thread 1:** Completes checkout

   - Status = BORROWED
   - `borrowedBy = alice`
   - Creates Lending record
   - Releases synchronized lock
   - Returns Lending object

4. **Thread 2:** Lock acquired, enters synchronized block
   - Check: `status == AVAILABLE` → false (status = BORROWED) ✗
   - `checkout()` returns null immediately
   - No state changes made
   - Returns null (checkout failed)

**Final State:**

```
PB-001: BORROWED by Alice (only one successful checkout)
Alice: 1 new loan
Bob: No new loan (checkout rejected)
Thread-safe behavior: Only one checkout succeeded
No data corruption, proper synchronization
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions & Invalid Inputs

- **Empty/Null ISBN or Barcode**: Validation in constructors
- **Member at Loan Limit**: `canBorrow()` returns false when `activeLoans.size() >= MAX_BOOKS_ALLOWED`
- **Member with Unpaid Fines**: `canBorrow()` returns false when `hasUnpaidFines()`
- **Checkout Already Borrowed Book**: `isAvailable()` returns false
- **Return Non-Borrowed Book**: `borrowedBy` is null, handled gracefully
- **Reserve Available Book**: System suggests checkout instead
- **Expired Library Card**: `isValid()` returns false
- **Zero or Negative Loan Period**: Validation in checkout method

### Concurrent Access Scenarios and Race Conditions

- **Double Checkout**: Two librarians try to checkout same book simultaneously
  - **Expected Behavior**: `synchronized` on `BookItem.checkout()` ensures only one succeeds
- **Concurrent Reservations**: Multiple members reserve same book
  - **Expected Behavior**: `ConcurrentLinkedQueue` maintains FIFO order
- **Return During Checkout**: Book returned while another checkout in progress
  - **Expected Behavior**: Status check in checkout prevents invalid state

### Unit Test Strategy for Each Class

**`Book`:**

- Test `addBookItem` creates and links BookItem correctly
- Test `hasAvailableCopy` with all available, some available, none available
- Test `getAvailableCopiesCount` accuracy

**`BookItem`:**

- Test `checkout` state transitions
- Test `checkout` fails when not AVAILABLE
- Test `returnItem` resets state
- Test `isOverdue` calculation

**`Member`:**

- Test `canBorrow` with various states (active, suspended, with fines, at limit)
- Test loan and reservation management
- Test fine tracking

**`Lending`:**

- Test `processReturn` with on-time return (no fine)
- Test `processReturn` with late return (fine calculated)
- Test `isOverdue` at various dates

**`BookCatalog`:**

- Test search by title, author, subject
- Test multi-word search (AND logic)
- Test case-insensitive search

### Integration Test Approach

- **Full Lending Cycle**: Checkout → Return → Verify state
- **Reservation Fulfillment**: Reserve → Return by another → Verify notification
- **Fine Workflow**: Late return → Fine created → Pay fine → Verify member can borrow
- **Search Integration**: Add books → Search → Verify results

### Load and Stress Testing Approach

- **Concurrent Checkouts**: 100 threads attempting checkouts simultaneously
- **Search Performance**: 10,000 books, measure search latency
- **Reservation Queue**: 1,000 reservations for same book, verify order

---

### Unit Tests

### Unit Tests

```java
// BookTest.java
public class BookTest {

    @Test
    void testAddBookItem() {
        Book book = new Book("978-0-13-468599-1", "Effective Java",
                            "Joshua Bloch", "Addison-Wesley", 2018,
                            BookFormat.PAPERBACK, "Programming", 416);

        BookItem item = book.addBookItem("EJ-001", 45.00, "A1-01");

        assertEquals(1, book.getTotalCopies());
        assertEquals(1, book.getAvailableCopiesCount());
        assertEquals(book, item.getBook());
    }

    @Test
    void testHasAvailableCopy() {
        Book book = createBookWithCopies(3);

        assertTrue(book.hasAvailableCopy());

        // Checkout all copies
        for (BookItem item : book.getBookItems()) {
            item.checkout(createMember(), 14);
        }

        assertFalse(book.hasAvailableCopy());
    }
}
```

```java
// BookItemTest.java
public class BookItemTest {

    @Test
    void testCheckoutChangesStatus() {
        BookItem item = createBookItem();
        Member member = createMember();

        assertTrue(item.isAvailable());

        boolean result = item.checkout(member, 14);

        assertTrue(result);
        assertFalse(item.isAvailable());
        assertEquals(BookItemStatus.BORROWED, item.getStatus());
        assertEquals(member, item.getBorrowedBy());
        assertNotNull(item.getDueDate());
    }

    @Test
    void testCannotCheckoutBorrowedItem() {
        BookItem item = createBookItem();
        Member member1 = createMember();
        Member member2 = createMember();

        item.checkout(member1, 14);

        boolean result = item.checkout(member2, 14);

        assertFalse(result);
        assertEquals(member1, item.getBorrowedBy());
    }

    @Test
    void testOverdueCalculation() {
        BookItem item = createBookItem();
        Member member = createMember();

        // Checkout with 0-day loan (immediately overdue)
        item.checkout(member, 0);

        // Simulate time passing
        // In real test, use time mocking

        assertTrue(item.isOverdue());
        assertTrue(item.getDaysOverdue() > 0);
    }
}
```

```java
// MemberTest.java
public class MemberTest {

    @Test
    void testCanBorrowWhenActive() {
        Member member = createActiveMember();

        assertTrue(member.canBorrow());
    }

    @Test
    void testCannotBorrowWithUnpaidFines() {
        Member member = createActiveMember();
        member.addFine(new Fine(null, 5.00));

        assertFalse(member.canBorrow());
    }

    @Test
    void testCannotBorrowAtLimit() {
        Member member = createActiveMember();

        // Add 5 loans (max limit)
        for (int i = 0; i < 5; i++) {
            member.addLoan(createLending());
        }

        assertFalse(member.canBorrow());
    }
}
```

```java
// BookCatalogTest.java
public class BookCatalogTest {

    private BookCatalog catalog;

    @BeforeEach
    void setUp() {
        catalog = new BookCatalog();
    }

    @Test
    void testSearchByTitle() {
        Book book1 = createBook("Effective Java");
        Book book2 = createBook("Java Concurrency");
        Book book3 = createBook("Python Programming");

        catalog.addBook(book1);
        catalog.addBook(book2);
        catalog.addBook(book3);

        List<Book> results = catalog.searchByTitle("Java");

        assertEquals(2, results.size());
        assertTrue(results.contains(book1));
        assertTrue(results.contains(book2));
    }

    @Test
    void testSearchByMultipleWords() {
        Book book1 = createBook("Effective Java");
        Book book2 = createBook("Java Concurrency in Practice");

        catalog.addBook(book1);
        catalog.addBook(book2);

        List<Book> results = catalog.searchByTitle("Effective Java");

        assertEquals(1, results.size());
        assertTrue(results.contains(book1));
    }
}
```

### Integration Tests

```java
// LibraryIntegrationTest.java
public class LibraryIntegrationTest {

    @Test
    void testFullLendingCycle() {
        Library.resetInstance();
        Library library = Library.getInstance("Test Library", "123 Test St");

        // Add book
        Book book = new Book("978-0-13-468599-1", "Effective Java",
                            "Joshua Bloch", "Addison-Wesley", 2018,
                            BookFormat.PAPERBACK, "Programming", 416);
        library.addBook(book);
        library.addBookCopy("978-0-13-468599-1", "EJ-001", 45.00, "A1-01");

        // Register member
        Member member = library.registerMember("Alice", "alice@test.com",
                                               "555-1234", "123 Oak St");

        // Checkout
        Lending lending = library.checkoutBook(member, "EJ-001");

        assertNotNull(lending);
        assertEquals(1, member.getBorrowedBooksCount());
        assertFalse(book.hasAvailableCopy());

        // Return
        Fine fine = library.returnBook("EJ-001");

        assertNull(fine);  // Not overdue
        assertEquals(0, member.getBorrowedBooksCount());
        assertTrue(book.hasAvailableCopy());
    }

    @Test
    void testReservationFulfillment() {
        Library.resetInstance();
        Library library = Library.getInstance("Test Library", "123 Test St");

        // Setup: 1 book, 1 copy
        Book book = createAndAddBook(library);
        library.addBookCopy(book.getIsbn(), "COPY-001", 45.00, "A1-01");

        Member alice = library.registerMember("Alice", "alice@test.com",
                                              "555-1234", "123 Oak St");
        Member bob = library.registerMember("Bob", "bob@test.com",
                                            "555-5678", "456 Pine St");

        // Alice checks out the only copy
        library.checkoutBook(alice, "COPY-001");

        // Bob tries to checkout, fails
        Lending bobLending = library.checkoutBook(bob, "COPY-001");
        assertNull(bobLending);

        // Bob reserves
        Reservation reservation = library.reserveBook(bob, book.getIsbn());
        assertNotNull(reservation);

        // Alice returns
        library.returnBook("COPY-001");

        // Bob should be notified (check reservation status)
        assertEquals(ReservationStatus.FULFILLED, reservation.getStatus());
    }
}
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.
