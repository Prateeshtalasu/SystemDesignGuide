# 🍽️ Restaurant Reservation System - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements
- Manage tables of different sizes and sections (indoor, outdoor, private)
- Handle booking with date, time, and party size
- Support waitlist when tables are unavailable
- Manage time slots and table turnover (2-hour slots)
- Handle booking modifications and cancellations
- Prevent double-booking
- Send confirmation notifications

### Explicit Out-of-Scope Items
- Online payment processing
- Menu management
- Order management
- Customer loyalty programs
- Staff scheduling
- Multi-restaurant chains

### Assumptions and Constraints
- **Single Restaurant**: One location
- **Fixed Time Slots**: 2-hour dining windows
- **Operating Hours**: 11 AM - 10 PM
- **Advance Booking**: Up to 30 days ahead
- **Party Size Limit**: Max 20 people

### Scale Assumptions (LLD Focus)
- Single restaurant with ~50 tables
- Peak reservations: ~200 per day
- Waitlist: Up to 50 entries per time slot
- In-memory storage acceptable

### Concurrency Model
- **Synchronized table assignment**: Prevents double-booking same table
- **ConcurrentHashMap** for reservations storage
- **Atomic availability check + assignment**: Single transaction
- **Waitlist processing**: Sequential to maintain fairness (FIFO)

### Public APIs
- `makeReservation(customer, dateTime, partySize)`: Create booking
- `cancelReservation(reservationId)`: Cancel booking
- `modifyReservation(reservationId, newDateTime)`: Change booking
- `getAvailableTables(dateTime, partySize)`: Check availability
- `joinWaitlist(customer, dateTime, partySize)`: Join waitlist

### Public API Usage Examples

```java
// Example 1: Basic usage
Restaurant restaurant = new Restaurant("The Fine Diner", Duration.ofMinutes(90));
ReservationService service = new ReservationService(restaurant);
Customer customer = new Customer("John Smith", "555-1234", "john@email.com");
LocalDate tomorrow = LocalDate.now().plusDays(1);
ReservationRequest request = new ReservationRequest(customer, tomorrow, LocalTime.of(19, 0), 2);
Reservation reservation = service.makeReservation(request);
System.out.println("Reservation confirmed: " + reservation.getId());

// Example 2: Typical workflow
Customer customer = new Customer("Jane Doe", "555-5678");
ReservationRequest request = new ReservationRequest(customer, tomorrow, LocalTime.of(19, 0), 4)
    .withSection("Patio")
    .withSpecialRequests("Birthday celebration");
try {
    Reservation reservation = service.makeReservation(request);
    System.out.println("Table assigned: " + reservation.getTable().getId());
} catch (NoTableAvailableException e) {
    WaitlistEntry entry = service.addToWaitlist(request);
    System.out.println("Added to waitlist: " + entry.getId());
}

// Example 3: Check availability and modify reservation
List<TimeSlot> availableSlots = service.getAvailableTimeSlots(tomorrow, 4);
System.out.println("Available time slots:");
for (TimeSlot slot : availableSlots) {
    System.out.println("  " + slot);
}

// Modify existing reservation
ReservationRequest newRequest = new ReservationRequest(
    customer, tomorrow, LocalTime.of(20, 0), 4);
Reservation modified = service.modifyReservation(reservation.getId(), newRequest);
System.out.println("Reservation modified: " + modified);

// Example 4: Cancel and waitlist notification
boolean cancelled = service.cancelReservation(reservation.getId());
if (cancelled) {
    System.out.println("Reservation cancelled. Waitlist customers will be notified.");
}
```

### Invariants
- **No Double Booking**: Table can have one reservation per time slot
- **Capacity Respect**: Party size ≤ table capacity
- **Time Validity**: Reservations within operating hours

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        RESTAURANT RESERVATION SYSTEM                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                      ReservationService                                   │   │
│  │                                                                           │   │
│  │  - restaurant: Restaurant                                                 │   │
│  │  - reservations: Map<String, Reservation>                                │   │
│  │  - waitlist: Map<LocalDate, Queue<WaitlistEntry>>                        │   │
│  │                                                                           │   │
│  │  + makeReservation(request): Reservation                                 │   │
│  │  + cancelReservation(id): boolean                                        │   │
│  │  + modifyReservation(id, request): Reservation                           │   │
│  │  + findAvailableTables(date, time, partySize): List<Table>              │   │
│  │  + addToWaitlist(request): WaitlistEntry                                 │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│                          │ manages                                               │
│                          ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                         Restaurant                                        │   │
│  │                                                                           │   │
│  │  - name: String                                                           │   │
│  │  - tables: List<Table>                                                   │   │
│  │  - sections: List<Section>                                               │   │
│  │  - openingHours: Map<DayOfWeek, TimeSlot>                               │   │
│  │  - slotDuration: Duration                                                │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│           ┌──────────────┼──────────────┐                                       │
│           │              │              │                                       │
│           ▼              ▼              ▼                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                             │
│  │    Table    │  │   Section   │  │  TimeSlot   │                             │
│  │             │  │             │  │             │                             │
│  │ - id        │  │ - name      │  │ - start     │                             │
│  │ - capacity  │  │ - tables    │  │ - end       │                             │
│  │ - section   │  │ - smoking   │  └─────────────┘                             │
│  └─────────────┘  └─────────────┘                                              │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                        Reservation                                        │   │
│  │                                                                           │   │
│  │  - id: String                     - status: ReservationStatus            │   │
│  │  - customer: Customer             - table: Table                          │   │
│  │  - date: LocalDate                - timeSlot: TimeSlot                   │   │
│  │  - partySize: int                 - specialRequests: String              │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌─────────────────┐         ┌─────────────────┐                                │
│  │    Customer     │         │  WaitlistEntry  │                                │
│  │                 │         │                 │                                │
│  │ - name          │         │ - customer      │                                │
│  │ - phone         │         │ - date          │                                │
│  │ - email         │         │ - partySize     │                                │
│  └─────────────────┘         │ - addedAt       │                                │
│                              └─────────────────┘                                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Time Slot Visualization

```
Restaurant Operating Hours: 11:00 AM - 10:00 PM
Slot Duration: 90 minutes

Time Slots:
┌─────────────────────────────────────────────────────────────────────────────┐
│ 11:00  12:30  14:00  15:30  17:00  18:30  20:00  21:30                     │
│   │      │      │      │      │      │      │      │                       │
│   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼                       │
│ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                   │
│ │Slot│ │Slot│ │Slot│ │Slot│ │Slot│ │Slot│ │Slot│ │Slot│                   │
│ │ 1  │ │ 2  │ │ 3  │ │ 4  │ │ 5  │ │ 6  │ │ 7  │ │ 8  │                   │
│ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘                   │
└─────────────────────────────────────────────────────────────────────────────┘

### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `Restaurant` | Restaurant configuration (tables, sections, operating hours) | Manages restaurant setup - encapsulates restaurant-level configuration and time slot generation |
| `Table` | Physical table properties (capacity, section) | Represents a table - encapsulates table-level properties and capacity checks |
| `Section` | Restaurant section properties (name, tables, smoking) | Groups tables by section - separates section-level properties from table properties |
| `TimeSlot` | Time period representation (start, end) | Encapsulates time period - separates time handling from reservation logic |
| `Customer` | Customer information (name, contact) | Stores customer data - separates customer information from reservation logic |
| `Reservation` | Reservation details (customer, table, time, status) | Encapsulates reservation data - stores all reservation information in one place |
| `ReservationRequest` | Reservation request parameters | Encapsulates reservation request - separates request data from reservation entity |
| `ReservationService` | Booking logic and reservation management | Handles booking workflow - separates business logic from domain objects, coordinates reservations |
| `WaitlistEntry` | Waitlist entry tracking | Tracks waitlist entries - separates waitlist management from reservation logic |

---

Table Availability for Dec 25:
┌─────────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│    Table    │11:00 │12:30 │14:00 │15:30 │17:00 │18:30 │20:00 │21:30 │
├─────────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│ T1 (2 ppl)  │  ✓   │  ✗   │  ✓   │  ✓   │  ✗   │  ✗   │  ✓   │  ✓   │
│ T2 (2 ppl)  │  ✓   │  ✓   │  ✓   │  ✗   │  ✓   │  ✗   │  ✓   │  ✓   │
│ T3 (4 ppl)  │  ✗   │  ✓   │  ✓   │  ✓   │  ✗   │  ✓   │  ✗   │  ✓   │
│ T4 (4 ppl)  │  ✓   │  ✓   │  ✗   │  ✓   │  ✓   │  ✗   │  ✓   │  ✓   │
│ T5 (6 ppl)  │  ✓   │  ✓   │  ✓   │  ✓   │  ✓   │  ✓   │  ✗   │  ✓   │
└─────────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Design the Time Model

```java
// Step 1: TimeSlot class
public class TimeSlot {
    private final LocalTime startTime;
    private final LocalTime endTime;
    
    public TimeSlot(LocalTime startTime, Duration duration) {
        this.startTime = startTime;
        this.endTime = startTime.plus(duration);
    }
    
    public boolean overlaps(TimeSlot other) {
        return startTime.isBefore(other.endTime) && 
               endTime.isAfter(other.startTime);
    }
}
```

---

### Phase 2: Design the Table Model

```java
// Step 2: Table class
public class Table {
    private final String id;
    private final int capacity;
    private final int minCapacity;
    private final Section section;
    
    public boolean canAccommodate(int partySize) {
        return partySize >= minCapacity && partySize <= capacity;
    }
}
```

---

### Phase 3: Design the Reservation

```java
// Step 3: Reservation class
public class Reservation {
    private final String id;
    private final Customer customer;
    private final LocalDate date;
    private final TimeSlot timeSlot;
    private final int partySize;
    private Table table;
    private ReservationStatus status;  // PENDING, CONFIRMED, SEATED, COMPLETED, CANCELLED, NO_SHOW
}
```

---

### Phase 4: Design the Reservation Service

```java
// Step 4: Reservation service
public class ReservationService {
    // Map: Date → Table → Set of booked TimeSlots
    private final Map<LocalDate, Map<Table, Set<TimeSlot>>> tableBookings;
    private final Map<String, Reservation> reservations;
    
    public synchronized Reservation makeReservation(ReservationRequest request) {
        validateRequest(request);
        
        TimeSlot timeSlot = restaurant.findTimeSlot(
            request.getDate(), request.getPreferredTime());
        
        Table table = findAvailableTable(
            request.getDate(), timeSlot, request.getPartySize());
        
        if (table == null) {
            throw new NoTableAvailableException("No table available");
        }
        
        Reservation reservation = new Reservation(
            request.getCustomer(), request.getDate(), 
            timeSlot, request.getPartySize(), table);
        
        bookTable(request.getDate(), table, timeSlot);
        reservations.put(reservation.getId(), reservation);
        
        return reservation;
    }
    
    private boolean isTableAvailable(LocalDate date, Table table, TimeSlot timeSlot) {
        Map<Table, Set<TimeSlot>> dayBookings = tableBookings.get(date);
        if (dayBookings == null) return true;
        
        Set<TimeSlot> bookedSlots = dayBookings.get(table);
        if (bookedSlots == null) return true;
        
        return !bookedSlots.contains(timeSlot);
    }
}
```

---

### Phase 5: Threading Model and Concurrency Control

**Threading Model:**

This restaurant reservation system handles **concurrent booking requests**:
- Multiple customers can try to book the same table/time slot
- Booking operations must be atomic to prevent double-booking
- Table availability checks must be thread-safe

**Concurrency Control:**

```java
// Synchronized method ensures atomic booking
public synchronized Reservation makeReservation(ReservationRequest request) {
    // Entire operation is synchronized:
    // 1. Check availability
    // 2. Find table
    // 3. Book table
    // All happens atomically
}

// Alternative: Per-table locking (better granularity)
private final Map<String, ReentrantLock> tableLocks = new ConcurrentHashMap<>();

public Reservation makeReservation(ReservationRequest request) {
    Table table = findAvailableTable(...);
    ReentrantLock lock = tableLocks.computeIfAbsent(
        table.getId(), k -> new ReentrantLock());
    
    lock.lock();
    try {
        // Double-check availability after acquiring lock
        if (!isTableAvailable(...)) {
            throw new NoTableAvailableException();
        }
        // Book table
        bookTable(...);
        return reservation;
    } finally {
        lock.unlock();
    }
}
```

**Why synchronized?**
- Without: Two threads can both see table as available and both book it
- With: Only one thread can check and book atomically

---

## STEP 2: Complete Java Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 Customer Class

```java
// Customer.java
package com.restaurant;

/**
 * Represents a customer making a reservation.
 */
public class Customer {
    
    private final String id;
    private final String name;
    private final String phone;
    private final String email;
    
    public Customer(String name, String phone) {
        this(name, phone, null);
    }
    
    public Customer(String name, String phone, String email) {
        this.id = java.util.UUID.randomUUID().toString().substring(0, 8);
        this.name = name;
        this.phone = phone;
        this.email = email;
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getPhone() { return phone; }
    public String getEmail() { return email; }
    
    @Override
    public String toString() {
        return String.format("Customer[%s, %s]", name, phone);
    }
}
```

### 2.2 TimeSlot Class

```java
// TimeSlot.java
package com.restaurant;

import java.time.LocalTime;
import java.time.Duration;

/**
 * Represents a time slot for reservations.
 */
public class TimeSlot {
    
    private final LocalTime startTime;
    private final LocalTime endTime;
    
    public TimeSlot(LocalTime startTime, LocalTime endTime) {
        if (startTime.isAfter(endTime)) {
            throw new IllegalArgumentException("Start time must be before end time");
        }
        this.startTime = startTime;
        this.endTime = endTime;
    }
    
    public TimeSlot(LocalTime startTime, Duration duration) {
        this(startTime, startTime.plus(duration));
    }
    
    /**
     * Checks if this time slot overlaps with another.
     */
    public boolean overlaps(TimeSlot other) {
        return startTime.isBefore(other.endTime) && endTime.isAfter(other.startTime);
    }
    
    /**
     * Checks if a time falls within this slot.
     */
    public boolean contains(LocalTime time) {
        return !time.isBefore(startTime) && time.isBefore(endTime);
    }
    
    public Duration getDuration() {
        return Duration.between(startTime, endTime);
    }
    
    // Getters
    public LocalTime getStartTime() { return startTime; }
    public LocalTime getEndTime() { return endTime; }
    
    @Override
    public String toString() {
        return String.format("%s-%s", startTime, endTime);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof TimeSlot)) return false;
        TimeSlot other = (TimeSlot) o;
        return startTime.equals(other.startTime) && endTime.equals(other.endTime);
    }
    
    @Override
    public int hashCode() {
        return startTime.hashCode() * 31 + endTime.hashCode();
    }
}
```

### 2.3 Section Class

```java
// Section.java
package com.restaurant;

/**
 * Represents a section of the restaurant (e.g., patio, main floor).
 */
public class Section {
    
    private final String id;
    private final String name;
    private final boolean outdoor;
    private final boolean smokingAllowed;
    
    public Section(String name) {
        this(name, false, false);
    }
    
    public Section(String name, boolean outdoor, boolean smokingAllowed) {
        this.id = java.util.UUID.randomUUID().toString().substring(0, 8);
        this.name = name;
        this.outdoor = outdoor;
        this.smokingAllowed = smokingAllowed;
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public boolean isOutdoor() { return outdoor; }
    public boolean isSmokingAllowed() { return smokingAllowed; }
    
    @Override
    public String toString() {
        return name;
    }
}
```

### 2.4 Table Class

```java
// Table.java
package com.restaurant;

/**
 * Represents a table in the restaurant.
 */
public class Table {
    
    private final String id;
    private final int tableNumber;
    private final int capacity;
    private final int minCapacity;
    private final Section section;
    
    public Table(int tableNumber, int capacity, Section section) {
        this(tableNumber, capacity, 1, section);
    }
    
    public Table(int tableNumber, int capacity, int minCapacity, Section section) {
        this.id = "T" + tableNumber;
        this.tableNumber = tableNumber;
        this.capacity = capacity;
        this.minCapacity = minCapacity;
        this.section = section;
    }
    
    /**
     * Checks if this table can accommodate the party size.
     */
    public boolean canAccommodate(int partySize) {
        return partySize >= minCapacity && partySize <= capacity;
    }
    
    // Getters
    public String getId() { return id; }
    public int getTableNumber() { return tableNumber; }
    public int getCapacity() { return capacity; }
    public int getMinCapacity() { return minCapacity; }
    public Section getSection() { return section; }
    
    @Override
    public String toString() {
        return String.format("Table %d (%d seats, %s)", tableNumber, capacity, section);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Table)) return false;
        return id.equals(((Table) o).id);
    }
    
    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

### 2.5 ReservationStatus Enum

```java
// ReservationStatus.java
package com.restaurant;

/**
 * Status of a reservation.
 */
public enum ReservationStatus {
    PENDING,      // Awaiting confirmation
    CONFIRMED,    // Confirmed by restaurant
    SEATED,       // Customer has arrived and seated
    COMPLETED,    // Dining completed
    CANCELLED,    // Cancelled by customer or restaurant
    NO_SHOW       // Customer didn't show up
}
```

### 2.6 Reservation Class

```java
// Reservation.java
package com.restaurant;

import java.time.LocalDate;
import java.time.LocalDateTime;

/**
 * Represents a table reservation.
 */
public class Reservation {
    
    private final String id;
    private final Customer customer;
    private final LocalDate date;
    private final TimeSlot timeSlot;
    private final int partySize;
    private Table table;
    private ReservationStatus status;
    private String specialRequests;
    private final LocalDateTime createdAt;
    
    public Reservation(Customer customer, LocalDate date, TimeSlot timeSlot, 
                       int partySize, Table table) {
        this.id = generateId();
        this.customer = customer;
        this.date = date;
        this.timeSlot = timeSlot;
        this.partySize = partySize;
        this.table = table;
        this.status = ReservationStatus.CONFIRMED;
        this.createdAt = LocalDateTime.now();
    }
    
    private String generateId() {
        return "RES-" + System.currentTimeMillis() % 100000;
    }
    
    public void cancel() {
        this.status = ReservationStatus.CANCELLED;
    }
    
    public void markSeated() {
        this.status = ReservationStatus.SEATED;
    }
    
    public void markCompleted() {
        this.status = ReservationStatus.COMPLETED;
    }
    
    public void markNoShow() {
        this.status = ReservationStatus.NO_SHOW;
    }
    
    public void setTable(Table table) {
        this.table = table;
    }
    
    public void setSpecialRequests(String requests) {
        this.specialRequests = requests;
    }
    
    public boolean isActive() {
        return status == ReservationStatus.CONFIRMED || 
               status == ReservationStatus.PENDING ||
               status == ReservationStatus.SEATED;
    }
    
    // Getters
    public String getId() { return id; }
    public Customer getCustomer() { return customer; }
    public LocalDate getDate() { return date; }
    public TimeSlot getTimeSlot() { return timeSlot; }
    public int getPartySize() { return partySize; }
    public Table getTable() { return table; }
    public ReservationStatus getStatus() { return status; }
    public String getSpecialRequests() { return specialRequests; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    
    @Override
    public String toString() {
        return String.format("Reservation[%s: %s on %s at %s, party of %d, %s, %s]",
                id, customer.getName(), date, timeSlot.getStartTime(), 
                partySize, table, status);
    }
}
```

### 2.7 ReservationRequest Class

```java
// ReservationRequest.java
package com.restaurant;

import java.time.LocalDate;
import java.time.LocalTime;

/**
 * Request to make a reservation.
 */
public class ReservationRequest {
    
    private final Customer customer;
    private final LocalDate date;
    private final LocalTime preferredTime;
    private final int partySize;
    private String preferredSection;
    private String specialRequests;
    
    public ReservationRequest(Customer customer, LocalDate date, 
                              LocalTime preferredTime, int partySize) {
        this.customer = customer;
        this.date = date;
        this.preferredTime = preferredTime;
        this.partySize = partySize;
    }
    
    public ReservationRequest withSection(String section) {
        this.preferredSection = section;
        return this;
    }
    
    public ReservationRequest withSpecialRequests(String requests) {
        this.specialRequests = requests;
        return this;
    }
    
    // Getters
    public Customer getCustomer() { return customer; }
    public LocalDate getDate() { return date; }
    public LocalTime getPreferredTime() { return preferredTime; }
    public int getPartySize() { return partySize; }
    public String getPreferredSection() { return preferredSection; }
    public String getSpecialRequests() { return specialRequests; }
}
```

### 2.8 WaitlistEntry Class

```java
// WaitlistEntry.java
package com.restaurant;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;

/**
 * Entry in the waitlist when no tables are available.
 */
public class WaitlistEntry {
    
    private final String id;
    private final Customer customer;
    private final LocalDate date;
    private final LocalTime preferredTime;
    private final int partySize;
    private final LocalDateTime addedAt;
    private boolean notified;
    
    public WaitlistEntry(Customer customer, LocalDate date, 
                         LocalTime preferredTime, int partySize) {
        this.id = "WL-" + System.currentTimeMillis() % 100000;
        this.customer = customer;
        this.date = date;
        this.preferredTime = preferredTime;
        this.partySize = partySize;
        this.addedAt = LocalDateTime.now();
        this.notified = false;
    }
    
    public void markNotified() {
        this.notified = true;
    }
    
    // Getters
    public String getId() { return id; }
    public Customer getCustomer() { return customer; }
    public LocalDate getDate() { return date; }
    public LocalTime getPreferredTime() { return preferredTime; }
    public int getPartySize() { return partySize; }
    public LocalDateTime getAddedAt() { return addedAt; }
    public boolean isNotified() { return notified; }
    
    @Override
    public String toString() {
        return String.format("Waitlist[%s: %s for %s at %s, party of %d]",
                id, customer.getName(), date, preferredTime, partySize);
    }
}
```

### 2.9 Restaurant Class

```java
// Restaurant.java
package com.restaurant;

import java.time.*;
import java.util.*;

/**
 * Represents a restaurant with tables and operating hours.
 */
public class Restaurant {
    
    private final String name;
    private final List<Table> tables;
    private final List<Section> sections;
    private final Map<DayOfWeek, TimeSlot> operatingHours;
    private final Duration slotDuration;
    
    public Restaurant(String name, Duration slotDuration) {
        this.name = name;
        this.tables = new ArrayList<>();
        this.sections = new ArrayList<>();
        this.operatingHours = new EnumMap<>(DayOfWeek.class);
        this.slotDuration = slotDuration;
        
        // Default operating hours: 11 AM - 10 PM
        TimeSlot defaultHours = new TimeSlot(LocalTime.of(11, 0), LocalTime.of(22, 0));
        for (DayOfWeek day : DayOfWeek.values()) {
            operatingHours.put(day, defaultHours);
        }
    }
    
    public void addSection(Section section) {
        sections.add(section);
    }
    
    public void addTable(Table table) {
        tables.add(table);
    }
    
    public void setOperatingHours(DayOfWeek day, LocalTime open, LocalTime close) {
        operatingHours.put(day, new TimeSlot(open, close));
    }
    
    /**
     * Gets all time slots for a given date.
     */
    public List<TimeSlot> getTimeSlots(LocalDate date) {
        TimeSlot hours = operatingHours.get(date.getDayOfWeek());
        if (hours == null) {
            return Collections.emptyList();
        }
        
        List<TimeSlot> slots = new ArrayList<>();
        LocalTime current = hours.getStartTime();
        
        while (current.plus(slotDuration).isBefore(hours.getEndTime()) ||
               current.plus(slotDuration).equals(hours.getEndTime())) {
            slots.add(new TimeSlot(current, slotDuration));
            current = current.plus(slotDuration);
        }
        
        return slots;
    }
    
    /**
     * Finds the time slot that contains the given time.
     */
    public TimeSlot findTimeSlot(LocalDate date, LocalTime time) {
        for (TimeSlot slot : getTimeSlots(date)) {
            if (slot.contains(time)) {
                return slot;
            }
        }
        return null;
    }
    
    /**
     * Gets tables that can accommodate the party size.
     */
    public List<Table> getTablesForPartySize(int partySize) {
        List<Table> suitable = new ArrayList<>();
        for (Table table : tables) {
            if (table.canAccommodate(partySize)) {
                suitable.add(table);
            }
        }
        // Sort by capacity (prefer smaller tables that fit)
        suitable.sort(Comparator.comparingInt(Table::getCapacity));
        return suitable;
    }
    
    public boolean isOpen(LocalDate date, LocalTime time) {
        TimeSlot hours = operatingHours.get(date.getDayOfWeek());
        return hours != null && hours.contains(time);
    }
    
    // Getters
    public String getName() { return name; }
    public List<Table> getTables() { return Collections.unmodifiableList(tables); }
    public List<Section> getSections() { return Collections.unmodifiableList(sections); }
    public Duration getSlotDuration() { return slotDuration; }
}
```

### 2.10 ReservationService Class

```java
// ReservationService.java
package com.restaurant;

import java.time.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Main service for managing reservations.
 */
public class ReservationService {
    
    private final Restaurant restaurant;
    private final Map<String, Reservation> reservations;
    private final Map<LocalDate, Map<Table, Set<TimeSlot>>> tableBookings;
    private final Map<LocalDate, Queue<WaitlistEntry>> waitlist;
    
    public ReservationService(Restaurant restaurant) {
        this.restaurant = restaurant;
        this.reservations = new ConcurrentHashMap<>();
        this.tableBookings = new ConcurrentHashMap<>();
        this.waitlist = new ConcurrentHashMap<>();
    }
    
    /**
     * Makes a reservation.
     */
    public synchronized Reservation makeReservation(ReservationRequest request) {
        // Validate request
        validateRequest(request);
        
        // Find time slot
        TimeSlot timeSlot = restaurant.findTimeSlot(request.getDate(), request.getPreferredTime());
        if (timeSlot == null) {
            throw new IllegalArgumentException("Invalid time: restaurant not open");
        }
        
        // Find available table
        Table table = findAvailableTable(request.getDate(), timeSlot, 
                                         request.getPartySize(), request.getPreferredSection());
        
        if (table == null) {
            throw new NoTableAvailableException(
                "No table available for " + request.getPartySize() + 
                " guests at " + request.getPreferredTime());
        }
        
        // Create reservation
        Reservation reservation = new Reservation(
            request.getCustomer(),
            request.getDate(),
            timeSlot,
            request.getPartySize(),
            table
        );
        
        if (request.getSpecialRequests() != null) {
            reservation.setSpecialRequests(request.getSpecialRequests());
        }
        
        // Book the table
        bookTable(request.getDate(), table, timeSlot);
        reservations.put(reservation.getId(), reservation);
        
        return reservation;
    }
    
    /**
     * Cancels a reservation.
     */
    public synchronized boolean cancelReservation(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null || !reservation.isActive()) {
            return false;
        }
        
        reservation.cancel();
        releaseTable(reservation.getDate(), reservation.getTable(), reservation.getTimeSlot());
        
        // Check waitlist
        notifyWaitlist(reservation.getDate(), reservation.getTimeSlot(), 
                       reservation.getTable().getCapacity());
        
        return true;
    }
    
    /**
     * Modifies a reservation.
     */
    public synchronized Reservation modifyReservation(String reservationId, 
                                                       ReservationRequest newRequest) {
        Reservation existing = reservations.get(reservationId);
        if (existing == null || !existing.isActive()) {
            throw new IllegalArgumentException("Reservation not found or not active");
        }
        
        // Try to make new reservation first
        Reservation newReservation;
        try {
            // Temporarily release old booking
            releaseTable(existing.getDate(), existing.getTable(), existing.getTimeSlot());
            
            newReservation = makeReservation(newRequest);
            
            // Cancel old reservation
            existing.cancel();
            
        } catch (Exception e) {
            // Restore old booking if new one fails
            bookTable(existing.getDate(), existing.getTable(), existing.getTimeSlot());
            throw e;
        }
        
        return newReservation;
    }
    
    /**
     * Adds customer to waitlist.
     */
    public WaitlistEntry addToWaitlist(ReservationRequest request) {
        WaitlistEntry entry = new WaitlistEntry(
            request.getCustomer(),
            request.getDate(),
            request.getPreferredTime(),
            request.getPartySize()
        );
        
        waitlist.computeIfAbsent(request.getDate(), k -> new LinkedList<>())
                .offer(entry);
        
        return entry;
    }
    
    /**
     * Gets all available time slots for a date and party size.
     */
    public List<TimeSlot> getAvailableTimeSlots(LocalDate date, int partySize) {
        List<TimeSlot> available = new ArrayList<>();
        
        for (TimeSlot slot : restaurant.getTimeSlots(date)) {
            if (findAvailableTable(date, slot, partySize, null) != null) {
                available.add(slot);
            }
        }
        
        return available;
    }
    
    /**
     * Gets reservations for a date.
     */
    public List<Reservation> getReservationsForDate(LocalDate date) {
        List<Reservation> result = new ArrayList<>();
        for (Reservation res : reservations.values()) {
            if (res.getDate().equals(date) && res.isActive()) {
                result.add(res);
            }
        }
        result.sort(Comparator.comparing(r -> r.getTimeSlot().getStartTime()));
        return result;
    }
    
    /**
     * Gets a reservation by ID.
     */
    public Reservation getReservation(String id) {
        return reservations.get(id);
    }
    
    // ==================== Private Helper Methods ====================
    
    private void validateRequest(ReservationRequest request) {
        if (request.getPartySize() <= 0) {
            throw new IllegalArgumentException("Party size must be positive");
        }
        if (request.getDate().isBefore(LocalDate.now())) {
            throw new IllegalArgumentException("Cannot book in the past");
        }
        if (!restaurant.isOpen(request.getDate(), request.getPreferredTime())) {
            throw new IllegalArgumentException("Restaurant is closed at that time");
        }
    }
    
    private Table findAvailableTable(LocalDate date, TimeSlot timeSlot, 
                                     int partySize, String preferredSection) {
        List<Table> suitableTables = restaurant.getTablesForPartySize(partySize);
        
        for (Table table : suitableTables) {
            // Check section preference
            if (preferredSection != null && 
                !table.getSection().getName().equalsIgnoreCase(preferredSection)) {
                continue;
            }
            
            // Check availability
            if (isTableAvailable(date, table, timeSlot)) {
                return table;
            }
        }
        
        // If no table in preferred section, try any section
        if (preferredSection != null) {
            return findAvailableTable(date, timeSlot, partySize, null);
        }
        
        return null;
    }
    
    private boolean isTableAvailable(LocalDate date, Table table, TimeSlot timeSlot) {
        Map<Table, Set<TimeSlot>> dayBookings = tableBookings.get(date);
        if (dayBookings == null) {
            return true;
        }
        
        Set<TimeSlot> bookedSlots = dayBookings.get(table);
        if (bookedSlots == null) {
            return true;
        }
        
        return !bookedSlots.contains(timeSlot);
    }
    
    private void bookTable(LocalDate date, Table table, TimeSlot timeSlot) {
        tableBookings.computeIfAbsent(date, k -> new ConcurrentHashMap<>())
                     .computeIfAbsent(table, k -> ConcurrentHashMap.newKeySet())
                     .add(timeSlot);
    }
    
    private void releaseTable(LocalDate date, Table table, TimeSlot timeSlot) {
        Map<Table, Set<TimeSlot>> dayBookings = tableBookings.get(date);
        if (dayBookings != null) {
            Set<TimeSlot> bookedSlots = dayBookings.get(table);
            if (bookedSlots != null) {
                bookedSlots.remove(timeSlot);
            }
        }
    }
    
    private void notifyWaitlist(LocalDate date, TimeSlot timeSlot, int tableCapacity) {
        Queue<WaitlistEntry> dateWaitlist = waitlist.get(date);
        if (dateWaitlist == null || dateWaitlist.isEmpty()) {
            return;
        }
        
        // Find first matching entry
        for (WaitlistEntry entry : dateWaitlist) {
            if (entry.getPartySize() <= tableCapacity && !entry.isNotified()) {
                entry.markNotified();
                System.out.println("📱 Notifying " + entry.getCustomer().getName() + 
                                  ": Table available for " + date + " at " + timeSlot);
                break;
            }
        }
    }
}
```

### 2.11 Custom Exception

```java
// NoTableAvailableException.java
package com.restaurant;

/**
 * Exception thrown when no table is available for a reservation.
 */
public class NoTableAvailableException extends RuntimeException {
    
    public NoTableAvailableException(String message) {
        super(message);
    }
}
```

### 2.12 Demo Application

```java
// ReservationDemo.java
package com.restaurant;

import java.time.*;

public class ReservationDemo {
    
    public static void main(String[] args) {
        System.out.println("=== RESTAURANT RESERVATION SYSTEM DEMO ===\n");
        
        // ==================== Setup Restaurant ====================
        System.out.println("===== SETTING UP RESTAURANT =====\n");
        
        Restaurant restaurant = new Restaurant("The Fine Diner", Duration.ofMinutes(90));
        
        // Add sections
        Section mainFloor = new Section("Main Floor");
        Section patio = new Section("Patio", true, false);
        Section privateRoom = new Section("Private Room");
        
        restaurant.addSection(mainFloor);
        restaurant.addSection(patio);
        restaurant.addSection(privateRoom);
        
        // Add tables
        restaurant.addTable(new Table(1, 2, mainFloor));
        restaurant.addTable(new Table(2, 2, mainFloor));
        restaurant.addTable(new Table(3, 4, mainFloor));
        restaurant.addTable(new Table(4, 4, mainFloor));
        restaurant.addTable(new Table(5, 6, mainFloor));
        restaurant.addTable(new Table(6, 4, patio));
        restaurant.addTable(new Table(7, 6, patio));
        restaurant.addTable(new Table(8, 10, privateRoom));
        
        System.out.println("Restaurant: " + restaurant.getName());
        System.out.println("Tables: " + restaurant.getTables().size());
        System.out.println("Slot duration: " + restaurant.getSlotDuration().toMinutes() + " minutes");
        
        ReservationService service = new ReservationService(restaurant);
        
        // ==================== Make Reservations ====================
        System.out.println("\n===== MAKING RESERVATIONS =====\n");
        
        LocalDate today = LocalDate.now();
        LocalDate tomorrow = today.plusDays(1);
        
        // Reservation 1
        Customer customer1 = new Customer("John Smith", "555-1234", "john@email.com");
        ReservationRequest request1 = new ReservationRequest(
            customer1, tomorrow, LocalTime.of(19, 0), 2);
        
        Reservation res1 = service.makeReservation(request1);
        System.out.println("Created: " + res1);
        
        // Reservation 2
        Customer customer2 = new Customer("Jane Doe", "555-5678");
        ReservationRequest request2 = new ReservationRequest(
            customer2, tomorrow, LocalTime.of(19, 0), 4)
            .withSection("Patio")
            .withSpecialRequests("Birthday celebration");
        
        Reservation res2 = service.makeReservation(request2);
        System.out.println("Created: " + res2);
        
        // Reservation 3 - Large party
        Customer customer3 = new Customer("Bob Wilson", "555-9999");
        ReservationRequest request3 = new ReservationRequest(
            customer3, tomorrow, LocalTime.of(18, 30), 8);
        
        Reservation res3 = service.makeReservation(request3);
        System.out.println("Created: " + res3);
        
        // ==================== Check Availability ====================
        System.out.println("\n===== CHECK AVAILABILITY =====\n");
        
        System.out.println("Available slots for party of 4 on " + tomorrow + ":");
        for (TimeSlot slot : service.getAvailableTimeSlots(tomorrow, 4)) {
            System.out.println("  " + slot);
        }
        
        // ==================== Waitlist ====================
        System.out.println("\n===== WAITLIST =====\n");
        
        // Try to book when no table available
        Customer customer4 = new Customer("Alice Brown", "555-4444");
        ReservationRequest request4 = new ReservationRequest(
            customer4, tomorrow, LocalTime.of(19, 0), 10);
        
        try {
            service.makeReservation(request4);
        } catch (NoTableAvailableException e) {
            System.out.println("No table available: " + e.getMessage());
            WaitlistEntry entry = service.addToWaitlist(request4);
            System.out.println("Added to waitlist: " + entry);
        }
        
        // ==================== Cancel and Notify Waitlist ====================
        System.out.println("\n===== CANCEL RESERVATION =====\n");
        
        System.out.println("Cancelling reservation: " + res3.getId());
        service.cancelReservation(res3.getId());
        System.out.println("Reservation cancelled. Status: " + res3.getStatus());
        
        // ==================== View Reservations ====================
        System.out.println("\n===== RESERVATIONS FOR " + tomorrow + " =====\n");
        
        for (Reservation res : service.getReservationsForDate(tomorrow)) {
            System.out.println("  " + res);
        }
        
        // ==================== Modify Reservation ====================
        System.out.println("\n===== MODIFY RESERVATION =====\n");
        
        System.out.println("Original: " + res1);
        
        ReservationRequest modifyRequest = new ReservationRequest(
            customer1, tomorrow, LocalTime.of(20, 0), 3);
        
        Reservation modified = service.modifyReservation(res1.getId(), modifyRequest);
        System.out.println("Modified: " + modified);
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 5: Simulation / Dry Run

### Scenario: Making a Reservation

```
Request: Party of 4, Dec 25 at 7:00 PM

Step 1: Validate Request
- Party size > 0? Yes (4)
- Date in future? Yes
- Restaurant open? Yes (11 AM - 10 PM)

Step 2: Find Time Slot
- 7:00 PM falls in slot 18:30-20:00
- TimeSlot = 18:30-20:00

Step 3: Find Available Table
- Get tables for party of 4: [T3(4), T4(4), T6(4), T5(6), T7(6), T8(10)]
- Check T3: tableBookings[Dec25][T3] contains 18:30-20:00? No → Available!
- Return T3

Step 4: Create Reservation
- ID: RES-12345
- Customer: John Smith
- Date: Dec 25
- TimeSlot: 18:30-20:00
- Table: T3
- Status: CONFIRMED

Step 5: Book Table
- tableBookings[Dec25][T3].add(18:30-20:00)

Return: Reservation RES-12345
```

---

## File Structure

```
com/restaurant/
├── Customer.java
├── TimeSlot.java
├── Section.java
├── Table.java
├── ReservationStatus.java
├── Reservation.java
├── ReservationRequest.java
├── WaitlistEntry.java
├── Restaurant.java
├── ReservationService.java
├── NoTableAvailableException.java
└── ReservationDemo.java
```

