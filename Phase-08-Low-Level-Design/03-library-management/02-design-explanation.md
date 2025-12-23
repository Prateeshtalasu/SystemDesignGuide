# 📚 Library Management System - Design Explanation

## SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Book` | Store book metadata | Book information format changes |
| `BookItem` | Track physical copy state | Copy tracking rules change |
| `Member` | Manage member data and limits | Member rules change |
| `Lending` | Track single borrowing transaction | Lending rules change |
| `Fine` | Track fine for one lending | Fine calculation changes |
| `Reservation` | Track single reservation | Reservation rules change |
| `BookCatalog` | Index and search books | Search functionality changes |
| `BookLendingService` | Handle checkout/return operations | Lending workflow changes |
| `ReservationService` | Handle reservation operations | Reservation workflow changes |
| `NotificationService` | Send notifications | Notification channels change |

**Key SRP Decision: Book vs BookItem**

```java
// Book: Represents the TITLE (metadata)
public class Book {
    private final String isbn;
    private final String title;
    private final String author;
    // Shared across all copies
}

// BookItem: Represents a PHYSICAL COPY
public class BookItem {
    private final String barcode;
    private BookItemStatus status;
    private Member borrowedBy;
    // Unique to each copy
}
```

**Why separate?**
- A library may have 10 copies of "Effective Java"
- All share the same title, author, ISBN
- Each has different barcode, status, location
- Without separation: duplicate metadata, harder updates

---

### 2. Open/Closed Principle (OCP)

**Adding New Book Formats:**

```java
// Just add to enum - no class changes needed
public enum BookFormat {
    HARDCOVER,
    PAPERBACK,
    AUDIOBOOK,
    EBOOK,
    MAGAZINE,
    NEWSPAPER,
    // New formats:
    BRAILLE,
    LARGE_PRINT
}
```

**Adding New Notification Channels:**

```java
// Strategy pattern for notifications
public interface NotificationChannel {
    void send(String to, String subject, String message);
}

public class EmailNotification implements NotificationChannel {
    @Override
    public void send(String to, String subject, String message) {
        // Send email
    }
}

public class SMSNotification implements NotificationChannel {
    @Override
    public void send(String to, String subject, String message) {
        // Send SMS
    }
}

// New channel - no changes to existing code
public class PushNotification implements NotificationChannel {
    @Override
    public void send(String to, String subject, String message) {
        // Send push notification
    }
}
```

**Adding New Fine Calculation Strategies:**

```java
public interface FineCalculator {
    double calculate(Lending lending, LocalDate returnDate);
}

public class StandardFineCalculator implements FineCalculator {
    private static final double RATE_PER_DAY = 0.50;
    
    @Override
    public double calculate(Lending lending, LocalDate returnDate) {
        long daysOverdue = ChronoUnit.DAYS.between(lending.getDueDate(), returnDate);
        return daysOverdue > 0 ? daysOverdue * RATE_PER_DAY : 0;
    }
}

public class GraduatedFineCalculator implements FineCalculator {
    @Override
    public double calculate(Lending lending, LocalDate returnDate) {
        long daysOverdue = ChronoUnit.DAYS.between(lending.getDueDate(), returnDate);
        if (daysOverdue <= 0) return 0;
        if (daysOverdue <= 7) return daysOverdue * 0.25;
        if (daysOverdue <= 14) return 7 * 0.25 + (daysOverdue - 7) * 0.50;
        return 7 * 0.25 + 7 * 0.50 + (daysOverdue - 14) * 1.00;
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**Testing LSP with Book Items:**

If we had different types of book items:

```java
public abstract class BookItem {
    public abstract boolean checkout(Member member, int days);
    public abstract void returnItem();
}

public class PhysicalBookItem extends BookItem {
    @Override
    public boolean checkout(Member member, int days) {
        // Physical checkout
        return true;
    }
}

public class DigitalBookItem extends BookItem {
    @Override
    public boolean checkout(Member member, int days) {
        // Digital checkout (no physical location)
        return true;
    }
}

// Client code works with any BookItem
public void processCheckout(BookItem item, Member member) {
    if (item.checkout(member, 14)) {
        System.out.println("Checkout successful");
    }
}
```

**LSP Violation to Avoid:**

```java
// BAD: Breaking the contract
public class ReferenceBookItem extends BookItem {
    @Override
    public boolean checkout(Member member, int days) {
        throw new UnsupportedOperationException(
            "Reference books cannot be checked out!");
    }
}
```

**Better approach:**

```java
// Use interface segregation
public interface Borrowable {
    boolean checkout(Member member, int days);
    void returnItem();
}

public interface Readable {
    void read();
}

// Regular books are borrowable
public class PhysicalBookItem implements Borrowable, Readable { }

// Reference books are only readable
public class ReferenceBookItem implements Readable { }
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design (Could Improve):**

```java
// Member has many responsibilities
public class Member {
    public boolean canBorrow() { }
    public boolean canReserve() { }
    public void addLoan(Lending lending) { }
    public void addReservation(Reservation reservation) { }
    public void addFine(Fine fine) { }
}
```

**Better with ISP:**

```java
public interface Borrower {
    boolean canBorrow();
    void addLoan(Lending lending);
    void removeLoan(Lending lending);
    List<Lending> getActiveLoans();
}

public interface Reserver {
    boolean canReserve();
    void addReservation(Reservation reservation);
    void removeReservation(Reservation reservation);
    List<Reservation> getActiveReservations();
}

public interface Fineable {
    void addFine(Fine fine);
    void payFine(Fine fine);
    boolean hasUnpaidFines();
    double getTotalUnpaidFines();
}

// Member implements all
public class Member implements Borrower, Reserver, Fineable {
    // Implementation
}

// Staff might only need Borrower
public class StaffMember implements Borrower {
    // No reservation or fine handling
}
```

**ISP for Search:**

```java
public interface TitleSearchable {
    List<Book> searchByTitle(String title);
}

public interface AuthorSearchable {
    List<Book> searchByAuthor(String author);
}

public interface SubjectSearchable {
    List<Book> searchBySubject(String subject);
}

public interface FullTextSearchable {
    List<Book> search(String query);
}

// Catalog implements all
public class BookCatalog implements 
    TitleSearchable, AuthorSearchable, SubjectSearchable, FullTextSearchable {
    // Implementation
}

// Simple search only needs title
public class SimpleCatalog implements TitleSearchable {
    // Only title search
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
public class BookLendingService {
    private final BookCatalog catalog;  // Concrete class
    private final NotificationService notificationService;  // Concrete class
}
```

**Better with DIP:**

```java
// Define abstractions
public interface Catalog {
    Book findByIsbn(String isbn);
    BookItem findByBarcode(String barcode);
}

public interface Notifier {
    void sendCheckoutConfirmation(Member member, Lending lending);
    void sendDueDateReminder(Member member, Lending lending);
}

// Service depends on abstractions
public class BookLendingService {
    private final Catalog catalog;
    private final Notifier notifier;
    
    public BookLendingService(Catalog catalog, Notifier notifier) {
        this.catalog = catalog;
        this.notifier = notifier;
    }
}

// Concrete implementations
public class BookCatalog implements Catalog { }
public class EmailNotificationService implements Notifier { }
public class SMSNotificationService implements Notifier { }
```

**Benefits:**
- Easy to swap catalog implementations (in-memory vs database)
- Easy to swap notification channels
- Easy to mock for testing

---

## Design Patterns Used

### 1. Singleton Pattern

**Where:** `Library` class

```java
public class Library {
    private static Library instance;
    
    private Library(String name, String address) { }
    
    public static synchronized Library getInstance(String name, String address) {
        if (instance == null) {
            instance = new Library(name, address);
        }
        return instance;
    }
}
```

**Why:**
- One library per system
- Central coordination point
- Shared state (catalog, members)

---

### 2. Repository Pattern

**Where:** `BookCatalog` and `MemberRegistry`

```java
public class BookCatalog {
    private final Map<String, Book> booksByIsbn;
    private final Map<String, BookItem> itemsByBarcode;
    
    public void addBook(Book book) { }
    public Book findByIsbn(String isbn) { }
    public BookItem findByBarcode(String barcode) { }
    public List<Book> searchByTitle(String title) { }
}
```

**Benefits:**
- Abstracts data storage
- Provides domain-specific query methods
- Easy to swap storage (in-memory → database)

---

### 3. Observer Pattern (Implicit)

**Where:** Reservation notifications

```java
// When book is returned, notify reservation holder
public void processBookReturn(Book book) {
    Queue<Reservation> queue = reservationsByBook.get(book.getIsbn());
    if (queue != null && !queue.isEmpty()) {
        Reservation next = queue.peek();
        next.fulfill();
        notificationService.sendReservationReady(next.getMember(), next);
    }
}
```

**Full Observer Implementation:**

```java
public interface BookReturnObserver {
    void onBookReturned(BookItem item);
}

public class ReservationService implements BookReturnObserver {
    @Override
    public void onBookReturned(BookItem item) {
        processBookReturn(item.getBook());
    }
}

public class BookLendingService {
    private final List<BookReturnObserver> observers = new ArrayList<>();
    
    public void addObserver(BookReturnObserver observer) {
        observers.add(observer);
    }
    
    public Fine returnBook(String barcode) {
        // ... return logic ...
        
        // Notify observers
        for (BookReturnObserver observer : observers) {
            observer.onBookReturned(bookItem);
        }
        
        return fine;
    }
}
```

---

### 4. Strategy Pattern

**Where:** Fine calculation (potential improvement)

```java
public interface FineCalculationStrategy {
    double calculateFine(LocalDate dueDate, LocalDate returnDate);
}

public class PerDayFineStrategy implements FineCalculationStrategy {
    private final double ratePerDay;
    
    public PerDayFineStrategy(double ratePerDay) {
        this.ratePerDay = ratePerDay;
    }
    
    @Override
    public double calculateFine(LocalDate dueDate, LocalDate returnDate) {
        long daysOverdue = ChronoUnit.DAYS.between(dueDate, returnDate);
        return daysOverdue > 0 ? daysOverdue * ratePerDay : 0;
    }
}

public class MaxCapFineStrategy implements FineCalculationStrategy {
    private final double ratePerDay;
    private final double maxFine;
    
    @Override
    public double calculateFine(LocalDate dueDate, LocalDate returnDate) {
        double fine = new PerDayFineStrategy(ratePerDay)
            .calculateFine(dueDate, returnDate);
        return Math.min(fine, maxFine);
    }
}
```

---

### 5. Factory Pattern

**Where:** Creating members and lending records

```java
public class MemberFactory {
    private static long memberCounter = 0;
    
    public static Member createMember(String name, String email, 
                                      String phone, String address) {
        String memberId = "MEM-" + (++memberCounter);
        return new Member(memberId, name, email, phone, address);
    }
    
    public static Member createStudentMember(String name, String email,
                                             String phone, String address,
                                             String studentId) {
        Member member = createMember(name, email, phone, address);
        // Set student-specific limits
        return member;
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Book Contains Status

```java
// Rejected
public class Book {
    private String isbn;
    private String title;
    private BookItemStatus status;  // Where does this go for multiple copies?
    private Member borrowedBy;
}
```

**Why rejected:**
- Can't track multiple copies
- Each copy has different status
- Violates normalization

---

### Alternative 2: Lending Inside Member

```java
// Rejected
public class Member {
    private List<BookItem> borrowedBooks;
    private Map<BookItem, LocalDate> dueDates;
    
    public void borrowBook(BookItem item) {
        borrowedBooks.add(item);
        dueDates.put(item, LocalDate.now().plusDays(14));
    }
}
```

**Why rejected:**
- Lending logic scattered
- Hard to track lending history
- Can't easily query all lendings

---

### Alternative 3: Single Search Method

```java
// Rejected
public List<Book> search(String query, SearchType type) {
    switch (type) {
        case TITLE:
            return searchByTitle(query);
        case AUTHOR:
            return searchByAuthor(query);
        // ...
    }
}
```

**Why rejected:**
- Violates OCP (new search types need modification)
- Less type-safe
- Harder to combine searches

---

### Alternative 4: No Reservation Queue

```java
// Rejected
public class Reservation {
    // Just store reservation, no queue
}
```

**Why rejected:**
- Can't handle multiple reservations for same book
- No fair ordering (first-come-first-served)
- Users don't know their position

---

## What Would Break Without Each Class

| Class | What Breaks |
|-------|-------------|
| `Book` | Can't represent book metadata |
| `BookItem` | Can't track individual copies |
| `Member` | Can't track who borrows what |
| `LibraryCard` | No membership validation |
| `Lending` | Can't track borrowing history |
| `Fine` | Can't collect overdue fees |
| `Reservation` | Can't handle book requests |
| `BookCatalog` | Can't search for books |
| `BookLendingService` | No checkout/return workflow |
| `ReservationService` | No reservation handling |
| `NotificationService` | Members not informed |

---

## Concurrency Considerations

### Thread-Safe Catalog

```java
public class BookCatalog {
    // Use ConcurrentHashMap for thread safety
    private final Map<String, Book> booksByIsbn = new ConcurrentHashMap<>();
    private final Map<String, BookItem> itemsByBarcode = new ConcurrentHashMap<>();
    
    // Use ConcurrentHashMap.newKeySet() for thread-safe sets
    private final Map<String, Set<Book>> booksByTitle = new ConcurrentHashMap<>();
}
```

### Concurrent Checkout Prevention

```java
public class BookLendingService {
    
    public synchronized Lending checkoutBook(Member member, String barcode) {
        // Synchronized to prevent double checkout
        BookItem bookItem = catalog.findByBarcode(barcode);
        
        if (!bookItem.isAvailable()) {
            return null;  // Already checked out
        }
        
        bookItem.checkout(member, DEFAULT_LOAN_PERIOD);
        // ...
    }
}
```

**Better: Fine-grained locking**

```java
public class BookItem {
    private final Object lock = new Object();
    
    public boolean checkout(Member member, int days) {
        synchronized (lock) {
            if (status != BookItemStatus.AVAILABLE) {
                return false;
            }
            status = BookItemStatus.BORROWED;
            this.borrowedBy = member;
            this.dueDate = LocalDate.now().plusDays(days);
            return true;
        }
    }
}
```

### Reservation Queue Thread Safety

```java
public class ReservationService {
    // Use ConcurrentLinkedQueue for thread-safe queue
    private final Map<String, Queue<Reservation>> reservationsByBook = 
        new ConcurrentHashMap<>();
    
    public Reservation reserveBook(Member member, String isbn) {
        // Atomic operation to add to queue
        reservationsByBook.computeIfAbsent(isbn, k -> new ConcurrentLinkedQueue<>())
                         .offer(reservation);
    }
}
```

---

## Extensibility Analysis

### Adding E-book Support

```java
public class EBook extends Book {
    private final String fileFormat;  // PDF, EPUB
    private final long fileSizeBytes;
    private final String downloadUrl;
}

public class EBookItem extends BookItem {
    private int simultaneousLoans;  // E-books can have multiple "copies"
    
    @Override
    public boolean checkout(Member member, int days) {
        if (currentLoans < simultaneousLoans) {
            currentLoans++;
            return true;
        }
        return false;
    }
}
```

### Adding Inter-Library Loan

```java
public class InterLibraryLoan extends Lending {
    private final Library sourceLibrary;
    private final Library destinationLibrary;
    private final LocalDate requestDate;
    
    @Override
    public Fine processReturn() {
        Fine fine = super.processReturn();
        // Notify source library
        sourceLibrary.receiveReturnedBook(this.getBookItem());
        return fine;
    }
}

public interface InterLibraryLoanService {
    InterLibraryLoan requestFromOtherLibrary(Member member, String isbn, Library source);
    void fulfillRequest(InterLibraryLoan loan);
}
```

### Adding Membership Tiers

```java
public enum MembershipTier {
    BASIC(5, 14, 0.50),      // 5 books, 14 days, $0.50/day fine
    PREMIUM(10, 21, 0.25),   // 10 books, 21 days, $0.25/day fine
    VIP(20, 30, 0.00);       // 20 books, 30 days, no fines
    
    private final int maxBooks;
    private final int loanPeriod;
    private final double fineRate;
    
    // Constructor and getters
}

public class Member {
    private MembershipTier tier;
    
    public boolean canBorrow() {
        return activeLoans.size() < tier.getMaxBooks();
    }
    
    public int getLoanPeriod() {
        return tier.getLoanPeriod();
    }
}
```

