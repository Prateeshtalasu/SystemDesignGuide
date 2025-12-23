# 📚 Library Management System - Complete Solution

## Problem Statement

Design a library management system that can:
- Manage books with multiple copies
- Handle member registration and management
- Support book lending and returns
- Implement search functionality (by title, author, ISBN)
- Handle reservations for unavailable books
- Calculate and manage fines for late returns
- Send notifications (due date reminders, availability alerts)

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           LIBRARY MANAGEMENT SYSTEM                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │    Library   │────────►│   BookCatalog    │────────►│      Book        │    │
│  │  (Singleton) │         │                  │         │                  │    │
│  └──────────────┘         └──────────────────┘         └────────┬─────────┘    │
│         │                                                       │              │
│         │                                                       │              │
│         ▼                                                       ▼              │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │MemberRegistry│────────►│     Member       │         │    BookItem      │    │
│  │              │         │                  │         │  (Physical Copy) │    │
│  └──────────────┘         └────────┬─────────┘         └────────┬─────────┘    │
│                                    │                            │              │
│                                    │                            │              │
│                                    ▼                            ▼              │
│                           ┌──────────────────┐         ┌──────────────────┐    │
│                           │   LibraryCard    │         │ BookItemStatus   │    │
│                           │                  │         │     (Enum)       │    │
│                           └──────────────────┘         └──────────────────┘    │
│                                                                                │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │ BookLending  │◄───────►│    Lending       │────────►│    Fine          │    │
│  │   Service    │         │                  │         │                  │    │
│  └──────────────┘         └──────────────────┘         └──────────────────┘    │
│         │                                                                       │
│         │                                                                       │
│         ▼                                                                       │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │ Reservation  │◄───────►│ReservationService│         │ Notification     │    │
│  │              │         │                  │         │    Service       │    │
│  └──────────────┘         └──────────────────┘         └──────────────────┘    │
│                                                                │              │
│                                                        ┌───────┴───────┐      │
│                                                        │               │      │
│                                                        ▼               ▼      │
│                                                 ┌──────────┐    ┌──────────┐  │
│                                                 │  Email   │    │   SMS    │  │
│                                                 │Notifier  │    │ Notifier │  │
│                                                 └──────────┘    └──────────┘  │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Relationships Summary

| Relationship | Type | Description |
|-------------|------|-------------|
| Library → BookCatalog | Composition | Library owns the catalog |
| Library → MemberRegistry | Composition | Library owns member registry |
| BookCatalog → Book | Aggregation | Catalog manages books |
| Book → BookItem | Composition | Book owns its physical copies |
| Member → LibraryCard | Composition | Member has a card |
| Member → Lending | Association | Member has active lendings |
| Lending → BookItem | Association | Lending references a book item |
| Lending → Fine | Composition | Lending may have associated fine |
| Reservation → Member | Association | Reservation is for a member |
| Reservation → Book | Association | Reservation is for a book |

---

## STEP 2: Complete Java Implementation

### 2.1 Enums

```java
// BookItemStatus.java
package com.library.enums;

/**
 * Status of a physical book copy (BookItem).
 */
public enum BookItemStatus {
    AVAILABLE,      // Can be borrowed
    BORROWED,       // Currently checked out
    RESERVED,       // Reserved for a member
    LOST,           // Reported lost
    DAMAGED,        // Not available due to damage
    BEING_REPAIRED  // Under repair
}
```

```java
// BookFormat.java
package com.library.enums;

/**
 * Format of the book.
 */
public enum BookFormat {
    HARDCOVER,
    PAPERBACK,
    AUDIOBOOK,
    EBOOK,
    MAGAZINE,
    NEWSPAPER
}
```

```java
// MemberStatus.java
package com.library.enums;

/**
 * Status of library membership.
 */
public enum MemberStatus {
    ACTIVE,         // Can borrow books
    SUSPENDED,      // Temporarily suspended (e.g., unpaid fines)
    EXPIRED,        // Membership expired
    BLACKLISTED     // Permanently banned
}
```

```java
// ReservationStatus.java
package com.library.enums;

/**
 * Status of a book reservation.
 */
public enum ReservationStatus {
    PENDING,        // Waiting for book to become available
    FULFILLED,      // Book is ready for pickup
    CANCELLED,      // Reservation cancelled
    EXPIRED         // Reservation expired (not picked up in time)
}
```

### 2.2 Book and BookItem Classes

```java
// Book.java
package com.library.models;

import com.library.enums.BookFormat;
import java.util.*;

/**
 * Represents a book title (not a physical copy).
 * 
 * A Book can have multiple BookItems (physical copies).
 * This separation allows tracking individual copies while
 * maintaining shared metadata.
 */
public class Book {
    
    private final String isbn;
    private final String title;
    private final String author;
    private final String publisher;
    private final int publicationYear;
    private final BookFormat format;
    private final String subject;
    private final int numberOfPages;
    
    // Physical copies of this book
    private final List<BookItem> bookItems;
    
    public Book(String isbn, String title, String author, String publisher,
                int publicationYear, BookFormat format, String subject, int numberOfPages) {
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
    
    /**
     * Adds a physical copy of this book.
     */
    public BookItem addBookItem(String barcode, double price, String rackLocation) {
        BookItem item = new BookItem(barcode, this, price, rackLocation);
        bookItems.add(item);
        return item;
    }
    
    /**
     * Gets all available copies.
     */
    public List<BookItem> getAvailableCopies() {
        return bookItems.stream()
                .filter(BookItem::isAvailable)
                .toList();
    }
    
    /**
     * Checks if any copy is available.
     */
    public boolean hasAvailableCopy() {
        return bookItems.stream().anyMatch(BookItem::isAvailable);
    }
    
    /**
     * Gets total number of copies.
     */
    public int getTotalCopies() {
        return bookItems.size();
    }
    
    /**
     * Gets number of available copies.
     */
    public int getAvailableCopiesCount() {
        return (int) bookItems.stream().filter(BookItem::isAvailable).count();
    }
    
    // Getters
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public String getPublisher() { return publisher; }
    public int getPublicationYear() { return publicationYear; }
    public BookFormat getFormat() { return format; }
    public String getSubject() { return subject; }
    public int getNumberOfPages() { return numberOfPages; }
    public List<BookItem> getBookItems() { return Collections.unmodifiableList(bookItems); }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Book book = (Book) obj;
        return isbn.equals(book.isbn);
    }
    
    @Override
    public int hashCode() {
        return isbn.hashCode();
    }
    
    @Override
    public String toString() {
        return String.format("Book[isbn=%s, title='%s', author='%s', available=%d/%d]",
                isbn, title, author, getAvailableCopiesCount(), getTotalCopies());
    }
}
```

```java
// BookItem.java
package com.library.models;

import com.library.enums.BookItemStatus;
import java.time.LocalDate;

/**
 * Represents a physical copy of a book.
 * 
 * Each BookItem has a unique barcode and tracks its own status,
 * location, and borrowing history.
 */
public class BookItem {
    
    private final String barcode;
    private final Book book;
    private final double price;
    private final LocalDate dateOfPurchase;
    
    private String rackLocation;
    private BookItemStatus status;
    private LocalDate dueDate;
    private Member borrowedBy;
    
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
    
    /**
     * Checks out this book item to a member.
     */
    public boolean checkout(Member member, int loanPeriodDays) {
        if (status != BookItemStatus.AVAILABLE) {
            return false;
        }
        
        this.status = BookItemStatus.BORROWED;
        this.borrowedBy = member;
        this.dueDate = LocalDate.now().plusDays(loanPeriodDays);
        return true;
    }
    
    /**
     * Returns this book item.
     */
    public void returnItem() {
        this.status = BookItemStatus.AVAILABLE;
        this.borrowedBy = null;
        this.dueDate = null;
    }
    
    /**
     * Reserves this book item.
     */
    public void reserve() {
        if (status == BookItemStatus.AVAILABLE) {
            this.status = BookItemStatus.RESERVED;
        }
    }
    
    /**
     * Cancels reservation.
     */
    public void cancelReservation() {
        if (status == BookItemStatus.RESERVED) {
            this.status = BookItemStatus.AVAILABLE;
        }
    }
    
    /**
     * Marks item as lost.
     */
    public void markAsLost() {
        this.status = BookItemStatus.LOST;
    }
    
    /**
     * Checks if item is available for borrowing.
     */
    public boolean isAvailable() {
        return status == BookItemStatus.AVAILABLE;
    }
    
    /**
     * Checks if item is overdue.
     */
    public boolean isOverdue() {
        return status == BookItemStatus.BORROWED && 
               dueDate != null && 
               LocalDate.now().isAfter(dueDate);
    }
    
    /**
     * Gets days overdue (0 if not overdue).
     */
    public long getDaysOverdue() {
        if (!isOverdue()) return 0;
        return java.time.temporal.ChronoUnit.DAYS.between(dueDate, LocalDate.now());
    }
    
    // Getters and setters
    public String getBarcode() { return barcode; }
    public Book getBook() { return book; }
    public double getPrice() { return price; }
    public LocalDate getDateOfPurchase() { return dateOfPurchase; }
    public String getRackLocation() { return rackLocation; }
    public void setRackLocation(String rackLocation) { this.rackLocation = rackLocation; }
    public BookItemStatus getStatus() { return status; }
    public LocalDate getDueDate() { return dueDate; }
    public Member getBorrowedBy() { return borrowedBy; }
    
    @Override
    public String toString() {
        return String.format("BookItem[barcode=%s, book='%s', status=%s, location=%s]",
                barcode, book.getTitle(), status, rackLocation);
    }
}
```

### 2.3 Member and LibraryCard Classes

```java
// Member.java
package com.library.models;

import com.library.enums.MemberStatus;
import java.time.LocalDate;
import java.util.*;

/**
 * Represents a library member.
 * 
 * Members can borrow books, make reservations, and have fines.
 */
public class Member {
    
    private final String memberId;
    private final String name;
    private final String email;
    private final String phone;
    private final String address;
    private final LocalDate memberSince;
    
    private MemberStatus status;
    private final LibraryCard libraryCard;
    
    // Current borrowings
    private final List<Lending> activeLoans;
    
    // Reservations
    private final List<Reservation> activeReservations;
    
    // Fines
    private final List<Fine> unpaidFines;
    
    // Limits
    private static final int MAX_BOOKS_ALLOWED = 5;
    private static final int MAX_RESERVATIONS_ALLOWED = 3;
    
    public Member(String memberId, String name, String email, String phone, String address) {
        this.memberId = memberId;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.address = address;
        this.memberSince = LocalDate.now();
        this.status = MemberStatus.ACTIVE;
        this.libraryCard = new LibraryCard(this);
        this.activeLoans = new ArrayList<>();
        this.activeReservations = new ArrayList<>();
        this.unpaidFines = new ArrayList<>();
    }
    
    /**
     * Checks if member can borrow more books.
     */
    public boolean canBorrow() {
        if (status != MemberStatus.ACTIVE) {
            return false;
        }
        if (hasUnpaidFines()) {
            return false;
        }
        return activeLoans.size() < MAX_BOOKS_ALLOWED;
    }
    
    /**
     * Checks if member can make more reservations.
     */
    public boolean canReserve() {
        if (status != MemberStatus.ACTIVE) {
            return false;
        }
        return activeReservations.size() < MAX_RESERVATIONS_ALLOWED;
    }
    
    /**
     * Adds a lending record.
     */
    public void addLoan(Lending lending) {
        activeLoans.add(lending);
    }
    
    /**
     * Removes a lending record (on return).
     */
    public void removeLoan(Lending lending) {
        activeLoans.remove(lending);
    }
    
    /**
     * Adds a reservation.
     */
    public void addReservation(Reservation reservation) {
        activeReservations.add(reservation);
    }
    
    /**
     * Removes a reservation.
     */
    public void removeReservation(Reservation reservation) {
        activeReservations.remove(reservation);
    }
    
    /**
     * Adds a fine.
     */
    public void addFine(Fine fine) {
        unpaidFines.add(fine);
    }
    
    /**
     * Pays a fine.
     */
    public void payFine(Fine fine) {
        fine.pay();
        unpaidFines.remove(fine);
    }
    
    /**
     * Checks if member has unpaid fines.
     */
    public boolean hasUnpaidFines() {
        return !unpaidFines.isEmpty();
    }
    
    /**
     * Gets total unpaid fine amount.
     */
    public double getTotalUnpaidFines() {
        return unpaidFines.stream()
                .mapToDouble(Fine::getAmount)
                .sum();
    }
    
    /**
     * Gets number of currently borrowed books.
     */
    public int getBorrowedBooksCount() {
        return activeLoans.size();
    }
    
    /**
     * Suspends the member.
     */
    public void suspend() {
        this.status = MemberStatus.SUSPENDED;
    }
    
    /**
     * Activates the member.
     */
    public void activate() {
        this.status = MemberStatus.ACTIVE;
    }
    
    // Getters
    public String getMemberId() { return memberId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    public String getAddress() { return address; }
    public LocalDate getMemberSince() { return memberSince; }
    public MemberStatus getStatus() { return status; }
    public LibraryCard getLibraryCard() { return libraryCard; }
    public List<Lending> getActiveLoans() { return Collections.unmodifiableList(activeLoans); }
    public List<Reservation> getActiveReservations() { return Collections.unmodifiableList(activeReservations); }
    public List<Fine> getUnpaidFines() { return Collections.unmodifiableList(unpaidFines); }
    
    @Override
    public String toString() {
        return String.format("Member[id=%s, name='%s', status=%s, loans=%d, fines=$%.2f]",
                memberId, name, status, activeLoans.size(), getTotalUnpaidFines());
    }
}
```

```java
// LibraryCard.java
package com.library.models;

import java.time.LocalDate;
import java.util.UUID;

/**
 * Represents a library card issued to a member.
 * 
 * The card has a unique number and expiration date.
 */
public class LibraryCard {
    
    private final String cardNumber;
    private final Member member;
    private final LocalDate issuedDate;
    private LocalDate expirationDate;
    private boolean isActive;
    
    private static final int VALIDITY_YEARS = 2;
    
    public LibraryCard(Member member) {
        this.cardNumber = generateCardNumber();
        this.member = member;
        this.issuedDate = LocalDate.now();
        this.expirationDate = issuedDate.plusYears(VALIDITY_YEARS);
        this.isActive = true;
    }
    
    private String generateCardNumber() {
        return "LIB-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
    
    /**
     * Checks if card is valid.
     */
    public boolean isValid() {
        return isActive && LocalDate.now().isBefore(expirationDate);
    }
    
    /**
     * Renews the card.
     */
    public void renew() {
        this.expirationDate = LocalDate.now().plusYears(VALIDITY_YEARS);
        this.isActive = true;
    }
    
    /**
     * Deactivates the card.
     */
    public void deactivate() {
        this.isActive = false;
    }
    
    // Getters
    public String getCardNumber() { return cardNumber; }
    public Member getMember() { return member; }
    public LocalDate getIssuedDate() { return issuedDate; }
    public LocalDate getExpirationDate() { return expirationDate; }
    public boolean isActive() { return isActive; }
    
    @Override
    public String toString() {
        return String.format("LibraryCard[number=%s, member=%s, valid=%s]",
                cardNumber, member.getName(), isValid());
    }
}
```

### 2.4 Lending and Fine Classes

```java
// Lending.java
package com.library.models;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

/**
 * Represents a book lending transaction.
 * 
 * Tracks when a book was borrowed, when it's due,
 * and when it was returned.
 */
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
    
    public Lending(BookItem bookItem, Member member) {
        this(bookItem, member, DEFAULT_LOAN_PERIOD_DAYS);
    }
    
    public Lending(BookItem bookItem, Member member, int loanPeriodDays) {
        this.lendingId = generateLendingId();
        this.bookItem = bookItem;
        this.member = member;
        this.borrowDate = LocalDate.now();
        this.dueDate = borrowDate.plusDays(loanPeriodDays);
        this.returnDate = null;
        this.fine = null;
    }
    
    private String generateLendingId() {
        return "LEND-" + System.currentTimeMillis();
    }
    
    /**
     * Processes the return of this book.
     * Calculates fine if overdue.
     */
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
    
    /**
     * Checks if this lending is overdue.
     */
    public boolean isOverdue() {
        LocalDate checkDate = (returnDate != null) ? returnDate : LocalDate.now();
        return checkDate.isAfter(dueDate);
    }
    
    /**
     * Gets days until due (negative if overdue).
     */
    public long getDaysUntilDue() {
        return ChronoUnit.DAYS.between(LocalDate.now(), dueDate);
    }
    
    /**
     * Extends the due date.
     */
    public boolean extend(int additionalDays) {
        if (isOverdue()) {
            return false;  // Can't extend overdue books
        }
        // In real system, would check if book is reserved by others
        return true;
    }
    
    // Getters
    public String getLendingId() { return lendingId; }
    public BookItem getBookItem() { return bookItem; }
    public Member getMember() { return member; }
    public LocalDate getBorrowDate() { return borrowDate; }
    public LocalDate getDueDate() { return dueDate; }
    public LocalDate getReturnDate() { return returnDate; }
    public Fine getFine() { return fine; }
    public boolean isReturned() { return returnDate != null; }
    
    @Override
    public String toString() {
        return String.format("Lending[id=%s, book='%s', member=%s, due=%s, returned=%s]",
                lendingId, bookItem.getBook().getTitle(), member.getName(), 
                dueDate, returnDate != null ? returnDate : "not yet");
    }
}
```

```java
// Fine.java
package com.library.models;

import java.time.LocalDate;

/**
 * Represents a fine for late book return.
 */
public class Fine {
    
    private final String fineId;
    private final Lending lending;
    private final double amount;
    private final LocalDate createdDate;
    
    private boolean paid;
    private LocalDate paidDate;
    
    public Fine(Lending lending, double amount) {
        this.fineId = "FINE-" + System.currentTimeMillis();
        this.lending = lending;
        this.amount = amount;
        this.createdDate = LocalDate.now();
        this.paid = false;
        this.paidDate = null;
    }
    
    /**
     * Marks the fine as paid.
     */
    public void pay() {
        this.paid = true;
        this.paidDate = LocalDate.now();
    }
    
    // Getters
    public String getFineId() { return fineId; }
    public Lending getLending() { return lending; }
    public double getAmount() { return amount; }
    public LocalDate getCreatedDate() { return createdDate; }
    public boolean isPaid() { return paid; }
    public LocalDate getPaidDate() { return paidDate; }
    
    @Override
    public String toString() {
        return String.format("Fine[id=%s, amount=$%.2f, paid=%s]",
                fineId, amount, paid);
    }
}
```

### 2.5 Reservation Class

```java
// Reservation.java
package com.library.models;

import com.library.enums.ReservationStatus;
import java.time.LocalDate;

/**
 * Represents a reservation for a book.
 * 
 * When a book is not available, members can reserve it.
 * They'll be notified when it becomes available.
 */
public class Reservation {
    
    private final String reservationId;
    private final Member member;
    private final Book book;
    private final LocalDate reservationDate;
    
    private ReservationStatus status;
    private LocalDate fulfillmentDate;
    private LocalDate expirationDate;
    
    // Reservation expires after 3 days of book becoming available
    private static final int FULFILLMENT_EXPIRY_DAYS = 3;
    
    public Reservation(Member member, Book book) {
        this.reservationId = "RES-" + System.currentTimeMillis();
        this.member = member;
        this.book = book;
        this.reservationDate = LocalDate.now();
        this.status = ReservationStatus.PENDING;
        this.fulfillmentDate = null;
        this.expirationDate = null;
    }
    
    /**
     * Marks reservation as fulfilled (book is available).
     */
    public void fulfill() {
        this.status = ReservationStatus.FULFILLED;
        this.fulfillmentDate = LocalDate.now();
        this.expirationDate = fulfillmentDate.plusDays(FULFILLMENT_EXPIRY_DAYS);
    }
    
    /**
     * Cancels the reservation.
     */
    public void cancel() {
        this.status = ReservationStatus.CANCELLED;
    }
    
    /**
     * Marks reservation as expired.
     */
    public void expire() {
        this.status = ReservationStatus.EXPIRED;
    }
    
    /**
     * Checks if reservation has expired.
     */
    public boolean isExpired() {
        if (status == ReservationStatus.FULFILLED && expirationDate != null) {
            return LocalDate.now().isAfter(expirationDate);
        }
        return status == ReservationStatus.EXPIRED;
    }
    
    /**
     * Checks if reservation is still active.
     */
    public boolean isActive() {
        return status == ReservationStatus.PENDING || 
               (status == ReservationStatus.FULFILLED && !isExpired());
    }
    
    // Getters
    public String getReservationId() { return reservationId; }
    public Member getMember() { return member; }
    public Book getBook() { return book; }
    public LocalDate getReservationDate() { return reservationDate; }
    public ReservationStatus getStatus() { return status; }
    public LocalDate getFulfillmentDate() { return fulfillmentDate; }
    public LocalDate getExpirationDate() { return expirationDate; }
    
    @Override
    public String toString() {
        return String.format("Reservation[id=%s, member=%s, book='%s', status=%s]",
                reservationId, member.getName(), book.getTitle(), status);
    }
}
```

### 2.6 Catalog and Search

```java
// BookCatalog.java
package com.library.catalog;

import com.library.models.Book;
import com.library.models.BookItem;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Manages the book catalog with search capabilities.
 * 
 * Provides multiple indexes for efficient searching.
 */
public class BookCatalog {
    
    // Primary index: ISBN -> Book
    private final Map<String, Book> booksByIsbn;
    
    // Secondary index: Barcode -> BookItem
    private final Map<String, BookItem> itemsByBarcode;
    
    // Search indexes
    private final Map<String, Set<Book>> booksByTitle;
    private final Map<String, Set<Book>> booksByAuthor;
    private final Map<String, Set<Book>> booksBySubject;
    
    public BookCatalog() {
        this.booksByIsbn = new ConcurrentHashMap<>();
        this.itemsByBarcode = new ConcurrentHashMap<>();
        this.booksByTitle = new ConcurrentHashMap<>();
        this.booksByAuthor = new ConcurrentHashMap<>();
        this.booksBySubject = new ConcurrentHashMap<>();
    }
    
    /**
     * Adds a book to the catalog.
     */
    public void addBook(Book book) {
        booksByIsbn.put(book.getIsbn(), book);
        
        // Index by title words
        indexByWords(book.getTitle(), book, booksByTitle);
        
        // Index by author
        indexByWords(book.getAuthor(), book, booksByAuthor);
        
        // Index by subject
        indexByWords(book.getSubject(), book, booksBySubject);
        
        // Index all book items
        for (BookItem item : book.getBookItems()) {
            itemsByBarcode.put(item.getBarcode(), item);
        }
        
        System.out.println("Added book to catalog: " + book.getTitle());
    }
    
    /**
     * Indexes a book by words in a field.
     */
    private void indexByWords(String text, Book book, Map<String, Set<Book>> index) {
        String[] words = text.toLowerCase().split("\\s+");
        for (String word : words) {
            index.computeIfAbsent(word, k -> ConcurrentHashMap.newKeySet())
                 .add(book);
        }
    }
    
    /**
     * Registers a book item (when added to a book).
     */
    public void registerBookItem(BookItem item) {
        itemsByBarcode.put(item.getBarcode(), item);
    }
    
    /**
     * Finds a book by ISBN.
     */
    public Book findByIsbn(String isbn) {
        return booksByIsbn.get(isbn);
    }
    
    /**
     * Finds a book item by barcode.
     */
    public BookItem findByBarcode(String barcode) {
        return itemsByBarcode.get(barcode);
    }
    
    /**
     * Searches books by title.
     */
    public List<Book> searchByTitle(String query) {
        return searchByField(query, booksByTitle);
    }
    
    /**
     * Searches books by author.
     */
    public List<Book> searchByAuthor(String query) {
        return searchByField(query, booksByAuthor);
    }
    
    /**
     * Searches books by subject.
     */
    public List<Book> searchBySubject(String query) {
        return searchByField(query, booksBySubject);
    }
    
    /**
     * Generic search by field.
     */
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
    
    /**
     * Full-text search across all fields.
     */
    public List<Book> search(String query) {
        Set<Book> results = new HashSet<>();
        results.addAll(searchByTitle(query));
        results.addAll(searchByAuthor(query));
        results.addAll(searchBySubject(query));
        return new ArrayList<>(results);
    }
    
    /**
     * Gets all books in the catalog.
     */
    public List<Book> getAllBooks() {
        return new ArrayList<>(booksByIsbn.values());
    }
    
    /**
     * Gets total number of unique titles.
     */
    public int getTotalBooks() {
        return booksByIsbn.size();
    }
    
    /**
     * Gets total number of physical copies.
     */
    public int getTotalBookItems() {
        return itemsByBarcode.size();
    }
}
```

### 2.7 Services

```java
// BookLendingService.java
package com.library.services;

import com.library.models.*;
import com.library.catalog.BookCatalog;
import java.util.*;

/**
 * Service for handling book lending operations.
 */
public class BookLendingService {
    
    private final BookCatalog catalog;
    private final NotificationService notificationService;
    private final Map<String, Lending> activeLendings;
    
    private static final int DEFAULT_LOAN_PERIOD = 14;
    
    public BookLendingService(BookCatalog catalog, NotificationService notificationService) {
        this.catalog = catalog;
        this.notificationService = notificationService;
        this.activeLendings = new HashMap<>();
    }
    
    /**
     * Checks out a book to a member.
     */
    public Lending checkoutBook(Member member, String barcode) {
        // Validate member
        if (!member.canBorrow()) {
            System.out.println("Member cannot borrow: " + member.getName());
            if (member.hasUnpaidFines()) {
                System.out.println("  Reason: Unpaid fines of $" + member.getTotalUnpaidFines());
            }
            return null;
        }
        
        // Find book item
        BookItem bookItem = catalog.findByBarcode(barcode);
        if (bookItem == null) {
            System.out.println("Book item not found: " + barcode);
            return null;
        }
        
        // Check availability
        if (!bookItem.isAvailable()) {
            System.out.println("Book item not available: " + bookItem.getStatus());
            return null;
        }
        
        // Perform checkout
        bookItem.checkout(member, DEFAULT_LOAN_PERIOD);
        
        // Create lending record
        Lending lending = new Lending(bookItem, member);
        activeLendings.put(lending.getLendingId(), lending);
        member.addLoan(lending);
        
        System.out.println("Checkout successful: " + bookItem.getBook().getTitle() + 
                          " to " + member.getName());
        System.out.println("  Due date: " + lending.getDueDate());
        
        // Send confirmation
        notificationService.sendCheckoutConfirmation(member, lending);
        
        return lending;
    }
    
    /**
     * Returns a book.
     */
    public Fine returnBook(String barcode) {
        BookItem bookItem = catalog.findByBarcode(barcode);
        if (bookItem == null) {
            System.out.println("Book item not found: " + barcode);
            return null;
        }
        
        Member member = bookItem.getBorrowedBy();
        if (member == null) {
            System.out.println("Book is not currently borrowed");
            return null;
        }
        
        // Find the lending record
        Lending lending = findLendingByBookItem(bookItem);
        if (lending == null) {
            System.out.println("Lending record not found");
            return null;
        }
        
        // Process return
        Fine fine = lending.processReturn();
        
        // Update book item
        bookItem.returnItem();
        
        // Update member
        member.removeLoan(lending);
        if (fine != null) {
            member.addFine(fine);
            System.out.println("Late return! Fine: $" + fine.getAmount());
        }
        
        // Remove from active lendings
        activeLendings.remove(lending.getLendingId());
        
        System.out.println("Return successful: " + bookItem.getBook().getTitle());
        
        return fine;
    }
    
    /**
     * Finds a lending by book item.
     */
    private Lending findLendingByBookItem(BookItem bookItem) {
        return activeLendings.values().stream()
                .filter(l -> l.getBookItem().equals(bookItem))
                .findFirst()
                .orElse(null);
    }
    
    /**
     * Renews a book (extends due date).
     */
    public boolean renewBook(String barcode) {
        BookItem bookItem = catalog.findByBarcode(barcode);
        if (bookItem == null) return false;
        
        Lending lending = findLendingByBookItem(bookItem);
        if (lending == null) return false;
        
        return lending.extend(DEFAULT_LOAN_PERIOD);
    }
    
    /**
     * Gets all overdue lendings.
     */
    public List<Lending> getOverdueLoans() {
        return activeLendings.values().stream()
                .filter(Lending::isOverdue)
                .toList();
    }
    
    /**
     * Sends reminders for books due soon.
     */
    public void sendDueDateReminders() {
        activeLendings.values().stream()
                .filter(l -> l.getDaysUntilDue() <= 3 && l.getDaysUntilDue() >= 0)
                .forEach(l -> notificationService.sendDueDateReminder(l.getMember(), l));
    }
}
```

```java
// ReservationService.java
package com.library.services;

import com.library.models.*;
import com.library.catalog.BookCatalog;
import com.library.enums.ReservationStatus;
import java.util.*;

/**
 * Service for handling book reservations.
 */
public class ReservationService {
    
    private final BookCatalog catalog;
    private final NotificationService notificationService;
    private final Map<String, Reservation> reservations;
    private final Map<String, Queue<Reservation>> reservationsByBook;
    
    public ReservationService(BookCatalog catalog, NotificationService notificationService) {
        this.catalog = catalog;
        this.notificationService = notificationService;
        this.reservations = new HashMap<>();
        this.reservationsByBook = new HashMap<>();
    }
    
    /**
     * Creates a reservation for a book.
     */
    public Reservation reserveBook(Member member, String isbn) {
        // Validate member
        if (!member.canReserve()) {
            System.out.println("Member cannot make more reservations");
            return null;
        }
        
        // Find book
        Book book = catalog.findByIsbn(isbn);
        if (book == null) {
            System.out.println("Book not found: " + isbn);
            return null;
        }
        
        // Check if book is available (no need to reserve)
        if (book.hasAvailableCopy()) {
            System.out.println("Book is available, no need to reserve");
            return null;
        }
        
        // Check if already reserved by this member
        if (hasActiveReservation(member, book)) {
            System.out.println("Member already has a reservation for this book");
            return null;
        }
        
        // Create reservation
        Reservation reservation = new Reservation(member, book);
        reservations.put(reservation.getReservationId(), reservation);
        
        // Add to book's reservation queue
        reservationsByBook.computeIfAbsent(isbn, k -> new LinkedList<>())
                         .offer(reservation);
        
        member.addReservation(reservation);
        
        System.out.println("Reservation created: " + reservation);
        
        return reservation;
    }
    
    /**
     * Cancels a reservation.
     */
    public void cancelReservation(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null) return;
        
        reservation.cancel();
        reservation.getMember().removeReservation(reservation);
        
        // Remove from book's queue
        Queue<Reservation> queue = reservationsByBook.get(reservation.getBook().getIsbn());
        if (queue != null) {
            queue.remove(reservation);
        }
        
        System.out.println("Reservation cancelled: " + reservationId);
    }
    
    /**
     * Processes book return - notifies next person in reservation queue.
     */
    public void processBookReturn(Book book) {
        Queue<Reservation> queue = reservationsByBook.get(book.getIsbn());
        if (queue == null || queue.isEmpty()) return;
        
        // Get next reservation
        Reservation nextReservation = queue.peek();
        while (nextReservation != null && 
               nextReservation.getStatus() != ReservationStatus.PENDING) {
            queue.poll();
            nextReservation = queue.peek();
        }
        
        if (nextReservation != null) {
            nextReservation.fulfill();
            notificationService.sendReservationReady(
                nextReservation.getMember(), nextReservation);
            System.out.println("Reservation fulfilled: " + nextReservation);
        }
    }
    
    /**
     * Checks if member has active reservation for book.
     */
    private boolean hasActiveReservation(Member member, Book book) {
        return member.getActiveReservations().stream()
                .anyMatch(r -> r.getBook().equals(book) && r.isActive());
    }
    
    /**
     * Gets position in reservation queue.
     */
    public int getQueuePosition(Reservation reservation) {
        Queue<Reservation> queue = reservationsByBook.get(
            reservation.getBook().getIsbn());
        if (queue == null) return -1;
        
        int position = 1;
        for (Reservation r : queue) {
            if (r.equals(reservation)) return position;
            if (r.getStatus() == ReservationStatus.PENDING) position++;
        }
        return -1;
    }
}
```

```java
// NotificationService.java
package com.library.services;

import com.library.models.*;

/**
 * Service for sending notifications to members.
 */
public class NotificationService {
    
    /**
     * Sends checkout confirmation.
     */
    public void sendCheckoutConfirmation(Member member, Lending lending) {
        String message = String.format(
            "Dear %s,\n\nYou have borrowed '%s'.\nDue date: %s\n\nThank you!",
            member.getName(),
            lending.getBookItem().getBook().getTitle(),
            lending.getDueDate()
        );
        sendEmail(member.getEmail(), "Book Checkout Confirmation", message);
    }
    
    /**
     * Sends due date reminder.
     */
    public void sendDueDateReminder(Member member, Lending lending) {
        String message = String.format(
            "Dear %s,\n\nReminder: '%s' is due on %s.\n" +
            "Please return it on time to avoid fines.\n\nThank you!",
            member.getName(),
            lending.getBookItem().getBook().getTitle(),
            lending.getDueDate()
        );
        sendEmail(member.getEmail(), "Book Due Date Reminder", message);
    }
    
    /**
     * Sends overdue notice.
     */
    public void sendOverdueNotice(Member member, Lending lending) {
        String message = String.format(
            "Dear %s,\n\n'%s' is overdue!\n" +
            "Due date was: %s\n" +
            "Please return it immediately.\n" +
            "Fine: $%.2f per day\n\nThank you!",
            member.getName(),
            lending.getBookItem().getBook().getTitle(),
            lending.getDueDate(),
            0.50
        );
        sendEmail(member.getEmail(), "OVERDUE NOTICE", message);
    }
    
    /**
     * Sends reservation ready notification.
     */
    public void sendReservationReady(Member member, Reservation reservation) {
        String message = String.format(
            "Dear %s,\n\n'%s' is now available for pickup!\n" +
            "Please pick it up by %s.\n\nThank you!",
            member.getName(),
            reservation.getBook().getTitle(),
            reservation.getExpirationDate()
        );
        sendEmail(member.getEmail(), "Reserved Book Available", message);
    }
    
    /**
     * Sends email (simulated).
     */
    private void sendEmail(String to, String subject, String body) {
        System.out.println("\n--- EMAIL ---");
        System.out.println("To: " + to);
        System.out.println("Subject: " + subject);
        System.out.println(body);
        System.out.println("--- END EMAIL ---\n");
    }
}
```

### 2.8 Library (Main Orchestrator)

```java
// Library.java
package com.library;

import com.library.catalog.BookCatalog;
import com.library.models.*;
import com.library.services.*;
import java.util.*;

/**
 * Main library class that orchestrates all operations.
 * 
 * PATTERN: Singleton - One library instance.
 */
public class Library {
    
    private static Library instance;
    
    private final String name;
    private final String address;
    
    private final BookCatalog catalog;
    private final Map<String, Member> members;
    private final BookLendingService lendingService;
    private final ReservationService reservationService;
    private final NotificationService notificationService;
    
    private Library(String name, String address) {
        this.name = name;
        this.address = address;
        this.catalog = new BookCatalog();
        this.members = new HashMap<>();
        this.notificationService = new NotificationService();
        this.lendingService = new BookLendingService(catalog, notificationService);
        this.reservationService = new ReservationService(catalog, notificationService);
    }
    
    public static synchronized Library getInstance(String name, String address) {
        if (instance == null) {
            instance = new Library(name, address);
        }
        return instance;
    }
    
    public static Library getInstance() {
        if (instance == null) {
            throw new IllegalStateException("Library not initialized");
        }
        return instance;
    }
    
    public static void resetInstance() {
        instance = null;
    }
    
    // ==================== Book Operations ====================
    
    public void addBook(Book book) {
        catalog.addBook(book);
    }
    
    public BookItem addBookCopy(String isbn, String barcode, double price, String location) {
        Book book = catalog.findByIsbn(isbn);
        if (book == null) {
            System.out.println("Book not found: " + isbn);
            return null;
        }
        
        BookItem item = book.addBookItem(barcode, price, location);
        catalog.registerBookItem(item);
        System.out.println("Added copy: " + barcode + " of " + book.getTitle());
        return item;
    }
    
    public List<Book> searchBooks(String query) {
        return catalog.search(query);
    }
    
    public List<Book> searchByTitle(String title) {
        return catalog.searchByTitle(title);
    }
    
    public List<Book> searchByAuthor(String author) {
        return catalog.searchByAuthor(author);
    }
    
    // ==================== Member Operations ====================
    
    public Member registerMember(String name, String email, String phone, String address) {
        String memberId = "MEM-" + System.currentTimeMillis();
        Member member = new Member(memberId, name, email, phone, address);
        members.put(memberId, member);
        System.out.println("Registered member: " + member);
        return member;
    }
    
    public Member findMember(String memberId) {
        return members.get(memberId);
    }
    
    // ==================== Lending Operations ====================
    
    public Lending checkoutBook(Member member, String barcode) {
        return lendingService.checkoutBook(member, barcode);
    }
    
    public Fine returnBook(String barcode) {
        Fine fine = lendingService.returnBook(barcode);
        
        // Check for reservations
        BookItem item = catalog.findByBarcode(barcode);
        if (item != null) {
            reservationService.processBookReturn(item.getBook());
        }
        
        return fine;
    }
    
    public boolean renewBook(String barcode) {
        return lendingService.renewBook(barcode);
    }
    
    // ==================== Reservation Operations ====================
    
    public Reservation reserveBook(Member member, String isbn) {
        return reservationService.reserveBook(member, isbn);
    }
    
    public void cancelReservation(String reservationId) {
        reservationService.cancelReservation(reservationId);
    }
    
    // ==================== Fine Operations ====================
    
    public void payFine(Member member, Fine fine) {
        member.payFine(fine);
        System.out.println("Fine paid: " + fine);
    }
    
    // ==================== Status ====================
    
    public void displayStatus() {
        System.out.println("\n========== LIBRARY STATUS ==========");
        System.out.println("Name: " + name);
        System.out.println("Address: " + address);
        System.out.println("Total Books: " + catalog.getTotalBooks());
        System.out.println("Total Copies: " + catalog.getTotalBookItems());
        System.out.println("Total Members: " + members.size());
        System.out.println("=====================================\n");
    }
    
    // Getters
    public String getName() { return name; }
    public String getAddress() { return address; }
    public BookCatalog getCatalog() { return catalog; }
}
```

### 2.9 Demo Application

```java
// LibraryDemo.java
package com.library;

import com.library.models.*;
import com.library.enums.*;
import java.util.List;

public class LibraryDemo {
    
    public static void main(String[] args) {
        Library.resetInstance();
        
        System.out.println("=== LIBRARY MANAGEMENT SYSTEM DEMO ===\n");
        
        // Create library
        Library library = Library.getInstance(
            "City Central Library",
            "123 Main Street"
        );
        
        // ==================== Add Books ====================
        System.out.println("===== ADDING BOOKS =====\n");
        
        Book book1 = new Book(
            "978-0-13-468599-1",
            "Effective Java",
            "Joshua Bloch",
            "Addison-Wesley",
            2018,
            BookFormat.PAPERBACK,
            "Programming Java",
            416
        );
        library.addBook(book1);
        
        // Add copies
        library.addBookCopy("978-0-13-468599-1", "EJ-001", 45.00, "A1-01");
        library.addBookCopy("978-0-13-468599-1", "EJ-002", 45.00, "A1-02");
        library.addBookCopy("978-0-13-468599-1", "EJ-003", 45.00, "A1-03");
        
        Book book2 = new Book(
            "978-0-59-651798-1",
            "Head First Design Patterns",
            "Eric Freeman",
            "O'Reilly",
            2020,
            BookFormat.PAPERBACK,
            "Design Patterns Programming",
            694
        );
        library.addBook(book2);
        library.addBookCopy("978-0-59-651798-1", "DP-001", 55.00, "A2-01");
        library.addBookCopy("978-0-59-651798-1", "DP-002", 55.00, "A2-02");
        
        // ==================== Register Members ====================
        System.out.println("\n===== REGISTERING MEMBERS =====\n");
        
        Member alice = library.registerMember(
            "Alice Johnson", "alice@email.com", "555-1234", "100 Oak St");
        Member bob = library.registerMember(
            "Bob Smith", "bob@email.com", "555-5678", "200 Pine St");
        
        // ==================== Search Books ====================
        System.out.println("\n===== SEARCHING BOOKS =====\n");
        
        System.out.println("Search for 'Java':");
        List<Book> results = library.searchBooks("Java");
        results.forEach(b -> System.out.println("  Found: " + b));
        
        System.out.println("\nSearch for 'Design':");
        results = library.searchBooks("Design");
        results.forEach(b -> System.out.println("  Found: " + b));
        
        // ==================== Checkout Books ====================
        System.out.println("\n===== CHECKING OUT BOOKS =====\n");
        
        Lending lending1 = library.checkoutBook(alice, "EJ-001");
        Lending lending2 = library.checkoutBook(alice, "DP-001");
        Lending lending3 = library.checkoutBook(bob, "EJ-002");
        
        System.out.println("\nAlice's loans: " + alice.getBorrowedBooksCount());
        System.out.println("Bob's loans: " + bob.getBorrowedBooksCount());
        
        // ==================== Try to checkout unavailable ====================
        System.out.println("\n===== RESERVATION SCENARIO =====\n");
        
        // All copies of Effective Java borrowed, Bob tries to get another
        Lending lending4 = library.checkoutBook(bob, "EJ-003");
        
        // Bob reserves the book
        Reservation reservation = library.reserveBook(bob, "978-0-13-468599-1");
        
        // ==================== Return Book ====================
        System.out.println("\n===== RETURNING BOOKS =====\n");
        
        // Alice returns - Bob should be notified
        Fine fine = library.returnBook("EJ-001");
        
        // ==================== Display Status ====================
        library.displayStatus();
        
        System.out.println("=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 5: Simulation / Dry Run

### Scenario: Complete Lending Cycle

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

## File Structure

```
com/library/
├── enums/
│   ├── BookItemStatus.java
│   ├── BookFormat.java
│   ├── MemberStatus.java
│   └── ReservationStatus.java
├── models/
│   ├── Book.java
│   ├── BookItem.java
│   ├── Member.java
│   ├── LibraryCard.java
│   ├── Lending.java
│   ├── Fine.java
│   └── Reservation.java
├── catalog/
│   └── BookCatalog.java
├── services/
│   ├── BookLendingService.java
│   ├── ReservationService.java
│   └── NotificationService.java
├── Library.java
└── LibraryDemo.java
```

