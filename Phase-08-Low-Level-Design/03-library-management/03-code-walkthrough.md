# 📚 Library Management System - Code Walkthrough

## Building From Scratch: Step-by-Step

### Phase 1: Define the Domain Model (Enums First)

```java
// Step 1: BookItemStatus enum
// File: com/library/enums/BookItemStatus.java

package com.library.enums;

public enum BookItemStatus {
    AVAILABLE,      // Ready to be borrowed
    BORROWED,       // Currently checked out
    RESERVED,       // Held for a member
    LOST,           // Reported missing
    DAMAGED,        // Not available due to damage
    BEING_REPAIRED  // Under maintenance
}
```

**Why these statuses?**
- `AVAILABLE`: The happy path, book can be borrowed
- `BORROWED`: Most common non-available state
- `RESERVED`: Prevents checkout by others when someone has reservation
- `LOST/DAMAGED/BEING_REPAIRED`: Operational states for inventory management

---

### Phase 2: Build Core Entities

```java
// Step 2: Book class (metadata)
// File: com/library/models/Book.java

package com.library.models;

import com.library.enums.BookFormat;
import java.util.*;

public class Book {
    
    // Immutable metadata (never changes for a book)
    private final String isbn;
    private final String title;
    private final String author;
    private final String publisher;
    private final int publicationYear;
    private final BookFormat format;
    private final String subject;
    private final int numberOfPages;
    
    // Mutable: physical copies can be added/removed
    private final List<BookItem> bookItems;
```

**Line-by-line explanation:**

- `final String isbn` - International Standard Book Number, unique identifier
- `final String title` - Book title, won't change
- `final List<BookItem> bookItems` - Composition: Book owns its copies

```java
    public Book(String isbn, String title, String author, String publisher,
                int publicationYear, BookFormat format, String subject, 
                int numberOfPages) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.publisher = publisher;
        this.publicationYear = publicationYear;
        this.format = format;
        this.subject = subject;
        this.numberOfPages = numberOfPages;
        this.bookItems = new ArrayList<>();
    }
```

**Why all parameters in constructor?**
- Book metadata is required and immutable
- No "half-created" books
- Ensures data integrity from creation

```java
    public BookItem addBookItem(String barcode, double price, String rackLocation) {
        BookItem item = new BookItem(barcode, this, price, rackLocation);
        bookItems.add(item);
        return item;
    }
```

**Why Book creates BookItem?**
- Enforces relationship: every BookItem belongs to a Book
- Can't create orphan BookItems
- Book manages its own collection

```java
    public List<BookItem> getAvailableCopies() {
        return bookItems.stream()
                .filter(BookItem::isAvailable)
                .toList();
    }
    
    public boolean hasAvailableCopy() {
        return bookItems.stream().anyMatch(BookItem::isAvailable);
    }
```

**Stream operations explained:**
- `filter(BookItem::isAvailable)` - Keep only items where `isAvailable()` returns true
- `toList()` - Collect to immutable list (Java 16+)
- `anyMatch()` - Returns true if any item matches predicate

---

### Phase 3: Build BookItem (Physical Copy)

```java
// Step 3: BookItem class
// File: com/library/models/BookItem.java

package com.library.models;

import com.library.enums.BookItemStatus;
import java.time.LocalDate;

public class BookItem {
    
    // Identity (immutable)
    private final String barcode;
    private final Book book;
    private final double price;
    private final LocalDate dateOfPurchase;
    
    // State (mutable)
    private String rackLocation;
    private BookItemStatus status;
    private LocalDate dueDate;
    private Member borrowedBy;
```

**Why separate identity and state?**
- Barcode never changes (printed on book)
- Price is purchase price (historical)
- Status changes frequently
- Location might change (reorganization)

```java
    public BookItem(String barcode, Book book, double price, String rackLocation) {
        this.barcode = barcode;
        this.book = book;
        this.price = price;
        this.rackLocation = rackLocation;
        this.dateOfPurchase = LocalDate.now();
        this.status = BookItemStatus.AVAILABLE;
        this.dueDate = null;
        this.borrowedBy = null;
    }
```

**Default state:**
- New books are AVAILABLE
- No due date (not borrowed)
- No borrower

```java
    public boolean checkout(Member member, int loanPeriodDays) {
        if (status != BookItemStatus.AVAILABLE) {
            return false;
        }
        
        this.status = BookItemStatus.BORROWED;
        this.borrowedBy = member;
        this.dueDate = LocalDate.now().plusDays(loanPeriodDays);
        return true;
    }
```

**State transition diagram:**

```
AVAILABLE ──checkout()──► BORROWED
    ▲                        │
    │                        │
    └────returnItem()────────┘
```

```java
    public boolean isOverdue() {
        return status == BookItemStatus.BORROWED && 
               dueDate != null && 
               LocalDate.now().isAfter(dueDate);
    }
    
    public long getDaysOverdue() {
        if (!isOverdue()) return 0;
        return java.time.temporal.ChronoUnit.DAYS.between(dueDate, LocalDate.now());
    }
```

**Date calculation:**
- `ChronoUnit.DAYS.between(start, end)` - Returns days between two dates
- Positive if end is after start

---

### Phase 4: Build Member System

```java
// Step 4: Member class
// File: com/library/models/Member.java

package com.library.models;

import com.library.enums.MemberStatus;
import java.time.LocalDate;
import java.util.*;

public class Member {
    
    // Identity
    private final String memberId;
    private final String name;
    private final String email;
    private final String phone;
    private final String address;
    private final LocalDate memberSince;
    
    // State
    private MemberStatus status;
    private final LibraryCard libraryCard;
    
    // Relationships
    private final List<Lending> activeLoans;
    private final List<Reservation> activeReservations;
    private final List<Fine> unpaidFines;
    
    // Business rules
    private static final int MAX_BOOKS_ALLOWED = 5;
    private static final int MAX_RESERVATIONS_ALLOWED = 3;
```

**Why constants for limits?**
- Easy to find and modify
- Could be moved to config
- Different member types could have different limits

```java
    public boolean canBorrow() {
        // Check 1: Member must be active
        if (status != MemberStatus.ACTIVE) {
            return false;
        }
        
        // Check 2: No unpaid fines
        if (hasUnpaidFines()) {
            return false;
        }
        
        // Check 3: Under loan limit
        return activeLoans.size() < MAX_BOOKS_ALLOWED;
    }
```

**Business rule validation:**
- All checks must pass
- Order matters for error messages
- Clear, readable conditions

```java
    public void addLoan(Lending lending) {
        activeLoans.add(lending);
    }
    
    public void removeLoan(Lending lending) {
        activeLoans.remove(lending);
    }
```

**Why Member manages loans?**
- Member knows their own loans
- Easy to check limits
- Bidirectional relationship with Lending

---

### Phase 5: Build Lending System

```java
// Step 5: Lending class
// File: com/library/models/Lending.java

package com.library.models;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public class Lending {
    
    private final String lendingId;
    private final BookItem bookItem;
    private final Member member;
    private final LocalDate borrowDate;
    private final LocalDate dueDate;
    
    private LocalDate returnDate;
    private Fine fine;
    
    private static final int DEFAULT_LOAN_PERIOD_DAYS = 14;
    private static final double FINE_PER_DAY = 0.50;
```

**Why Lending is a separate class?**
- Represents a transaction (event)
- Has its own lifecycle
- Tracks history (even after return)
- Can have associated fine

```java
    public Fine processReturn() {
        this.returnDate = LocalDate.now();
        
        if (isOverdue()) {
            long daysOverdue = ChronoUnit.DAYS.between(dueDate, returnDate);
            double fineAmount = daysOverdue * FINE_PER_DAY;
            this.fine = new Fine(this, fineAmount);
            return this.fine;
        }
        
        return null;
    }
```

**Return processing:**
1. Record return date
2. Check if overdue
3. Calculate fine if overdue
4. Create Fine object
5. Return fine (or null if on time)

**Example calculation:**
```
Due date: 2024-01-15
Return date: 2024-01-20
Days overdue: 5
Fine: 5 × $0.50 = $2.50
```

---

### Phase 6: Build Search Catalog

```java
// Step 6: BookCatalog class
// File: com/library/catalog/BookCatalog.java

package com.library.catalog;

import com.library.models.Book;
import com.library.models.BookItem;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class BookCatalog {
    
    // Primary indexes
    private final Map<String, Book> booksByIsbn;
    private final Map<String, BookItem> itemsByBarcode;
    
    // Search indexes (inverted indexes)
    private final Map<String, Set<Book>> booksByTitle;
    private final Map<String, Set<Book>> booksByAuthor;
    private final Map<String, Set<Book>> booksBySubject;
```

**Why multiple indexes?**
- `booksByIsbn`: O(1) lookup by ISBN
- `itemsByBarcode`: O(1) lookup by barcode
- `booksByTitle`: Fast search by title words
- Inverted index: word → set of books containing that word

```java
    public void addBook(Book book) {
        booksByIsbn.put(book.getIsbn(), book);
        
        // Index by title words
        indexByWords(book.getTitle(), book, booksByTitle);
        
        // Index by author
        indexByWords(book.getAuthor(), book, booksByAuthor);
        
        // Index by subject
        indexByWords(book.getSubject(), book, booksBySubject);
    }
    
    private void indexByWords(String text, Book book, Map<String, Set<Book>> index) {
        String[] words = text.toLowerCase().split("\\s+");
        for (String word : words) {
            index.computeIfAbsent(word, k -> ConcurrentHashMap.newKeySet())
                 .add(book);
        }
    }
```

**Indexing example:**
```
Book: "Effective Java" by Joshua Bloch

booksByTitle:
  "effective" → {Book1}
  "java" → {Book1}

booksByAuthor:
  "joshua" → {Book1}
  "bloch" → {Book1}
```

```java
    public List<Book> searchByTitle(String query) {
        return searchByField(query, booksByTitle);
    }
    
    private List<Book> searchByField(String query, Map<String, Set<Book>> index) {
        String[] words = query.toLowerCase().split("\\s+");
        
        Set<Book> results = null;
        
        for (String word : words) {
            Set<Book> matches = index.getOrDefault(word, Collections.emptySet());
            
            if (results == null) {
                results = new HashSet<>(matches);
            } else {
                results.retainAll(matches);  // Intersection
            }
        }
        
        return results != null ? new ArrayList<>(results) : Collections.emptyList();
    }
```

**Search algorithm (AND search):**
1. Split query into words
2. For each word, get matching books
3. Intersect results (book must match ALL words)

**Example:**
```
Query: "effective java"
Word "effective": {Book1, Book5}
Word "java": {Book1, Book3, Book7}
Intersection: {Book1}
```

---

### Phase 7: Build Services

```java
// Step 7: BookLendingService
// File: com/library/services/BookLendingService.java

public class BookLendingService {
    
    private final BookCatalog catalog;
    private final NotificationService notificationService;
    private final Map<String, Lending> activeLendings;
    
    public Lending checkoutBook(Member member, String barcode) {
        // Step 1: Validate member
        if (!member.canBorrow()) {
            System.out.println("Member cannot borrow: " + member.getName());
            if (member.hasUnpaidFines()) {
                System.out.println("  Reason: Unpaid fines");
            }
            return null;
        }
        
        // Step 2: Find book item
        BookItem bookItem = catalog.findByBarcode(barcode);
        if (bookItem == null) {
            System.out.println("Book item not found: " + barcode);
            return null;
        }
        
        // Step 3: Check availability
        if (!bookItem.isAvailable()) {
            System.out.println("Book not available: " + bookItem.getStatus());
            return null;
        }
        
        // Step 4: Perform checkout
        bookItem.checkout(member, DEFAULT_LOAN_PERIOD);
        
        // Step 5: Create lending record
        Lending lending = new Lending(bookItem, member);
        activeLendings.put(lending.getLendingId(), lending);
        member.addLoan(lending);
        
        // Step 6: Send notification
        notificationService.sendCheckoutConfirmation(member, lending);
        
        return lending;
    }
}
```

**Checkout flow diagram:**

```
checkoutBook(member, barcode)
         │
         ▼
    ┌────────────┐     No
    │ canBorrow? │─────────► Return null
    └─────┬──────┘
          │ Yes
          ▼
    ┌────────────┐     No
    │ Item found?│─────────► Return null
    └─────┬──────┘
          │ Yes
          ▼
    ┌────────────┐     No
    │ Available? │─────────► Return null
    └─────┬──────┘
          │ Yes
          ▼
    ┌────────────┐
    │  Checkout  │
    └─────┬──────┘
          │
          ▼
    ┌────────────┐
    │Create Lend │
    └─────┬──────┘
          │
          ▼
    ┌────────────┐
    │  Notify    │
    └─────┬──────┘
          │
          ▼
      Return lending
```

---

## Testing Approach

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

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `findByIsbn()` | O(1) | HashMap lookup |
| `findByBarcode()` | O(1) | HashMap lookup |
| `searchByTitle()` | O(W + R) | W = words in query, R = results |
| `addBook()` | O(W) | W = words to index |
| `checkoutBook()` | O(1) | Direct lookups |
| `returnBook()` | O(L) | L = active lendings |
| `reserveBook()` | O(1) | Queue operations |

### Space Complexity

| Data Structure | Space | Purpose |
|----------------|-------|---------|
| `booksByIsbn` | O(B) | B = number of books |
| `itemsByBarcode` | O(I) | I = number of items |
| `booksByTitle` | O(W × B) | W = unique words |
| `activeLendings` | O(L) | L = active lendings |
| `reservationsByBook` | O(R) | R = reservations |

### Bottlenecks at Scale

**1 million books:**
- Search indexes become large
- Solution: Use proper search engine (Elasticsearch)

**10,000 concurrent checkouts:**
- Single lock becomes bottleneck
- Solution: Partition by book category or location

---

## Interview Follow-ups with Answers

### Q1: How would you handle multiple library branches?

```java
public class LibraryBranch {
    private final String branchId;
    private final String name;
    private final String address;
    private final BookCatalog localCatalog;
    private final Map<String, Member> localMembers;
}

public class LibraryNetwork {
    private final List<LibraryBranch> branches;
    private final BookCatalog centralCatalog;  // All books
    
    public List<BookItem> findAvailableAcrossBranches(String isbn) {
        return branches.stream()
            .flatMap(b -> b.getLocalCatalog().findByIsbn(isbn)
                          .getAvailableCopies().stream())
            .toList();
    }
}
```

### Q2: How would you implement a recommendation system?

```java
public class RecommendationService {
    
    public List<Book> getRecommendations(Member member) {
        // Based on borrowing history
        Set<String> borrowedSubjects = member.getBorrowingHistory().stream()
            .map(l -> l.getBookItem().getBook().getSubject())
            .collect(Collectors.toSet());
        
        // Find books in same subjects, not yet borrowed
        return catalog.getAllBooks().stream()
            .filter(b -> borrowedSubjects.contains(b.getSubject()))
            .filter(b -> !member.hasBorrowed(b))
            .limit(10)
            .toList();
    }
}
```

### Q3: How would you handle digital books (e-books)?

```java
public class EBook extends Book {
    private final String fileFormat;
    private final int maxSimultaneousLoans;
    private int currentLoans;
    
    public boolean canLend() {
        return currentLoans < maxSimultaneousLoans;
    }
}

public class DigitalLending extends Lending {
    private final String downloadLink;
    private final LocalDateTime linkExpiration;
    
    @Override
    public Fine processReturn() {
        // Digital returns are automatic
        this.returnDate = LocalDate.now();
        ((EBook) getBookItem().getBook()).decrementLoans();
        return null;  // No fines for digital
    }
}
```

### Q4: How would you implement a waitlist with estimated wait times?

```java
public class WaitlistService {
    
    public WaitlistInfo getWaitlistInfo(Member member, Book book) {
        Queue<Reservation> queue = reservationsByBook.get(book.getIsbn());
        
        int position = getQueuePosition(member, queue);
        int avgLoanDays = calculateAverageLoanDays(book);
        int copiesCount = book.getTotalCopies();
        
        // Estimate: position / copies * average loan days
        int estimatedDays = (position / copiesCount) * avgLoanDays;
        
        return new WaitlistInfo(position, estimatedDays);
    }
}
```

### Q5: What would you do differently with more time?

1. **Add audit logging** - Track all operations for compliance
2. **Add analytics** - Popular books, peak times, member behavior
3. **Add caching** - Cache frequent searches
4. **Add batch operations** - Bulk import books
5. **Add API layer** - REST endpoints for external integration
6. **Add event sourcing** - Track all state changes
7. **Add full-text search** - Elasticsearch integration

